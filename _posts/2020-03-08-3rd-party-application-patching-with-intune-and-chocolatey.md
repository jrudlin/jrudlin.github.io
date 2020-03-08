---
layout: post
title: 3rd party Win10 application patching with Intune, Chocolatey and PSADT
subtitle: Keep third party apps updated/patched using the power of Chocolately combined with user interaction from the PowerShell App Deployment Toolkit, deployed through Intune.
share-img: "assets/images/ChocoAppPatching5.jpg"
date: 2020-03-08 20:48:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Windows 10
- Intune
- Chocolatey
- PowerShell App Deployment Toolkit
tags:
- Windows 10
- Intune
- Chocolatey
- PowerShell App Deployment Toolkit
- Scheduled Task
- PSADT
- Applications
- Upgrades
- Patching
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![ChocoPatching5]({{ site.baseurl }}/assets/images/ChocoAppPatching5.jpg)

Keep third party apps updated/patched using the power of Chocolately combined with user interaction from the PowerShell App Deployment Toolkit and ServiceUI, deployed through Intune.

* TOC
{:toc}

# Overview

Third party application patching (Adobe Reader for example) with Intune standalone is no mean feat without ConfigMgr Co-Management. Reporting (inventory) in Intune is still limited and there is no third party catalog import ability (CAB files) like in ConfigMgr, so usually custom solutions are needed if embarking on a modern desktop implementation. However, there are some 3rd party solutions now available if you want a non-Microsoft based solution.

In this short post, I wanted to show you how I kept applications updated and patched with the [Chocolatey](https://chocolatey.org/) Nuget package manager (Free version).
> Note: If you use the Business edition (paid) or better, then you can also manage and update applications installed outside of Chocolatey - so ones that the IT folk with local admin have installed manually!

These are the core components in use in this solution:
* [Intune - Win32 app deployment](https://docs.microsoft.com/en-us/intune/apps/apps-win32-app-management)
* Scheduled Task
* [PowerShell App Deployment Toolkit (PSADT)](https://psappdeploytoolkit.com)
* [Chocolatey - (Free license)](https://chocolatey.org/pricing)
* [Microsoft Deployment Toolkit](https://docs.microsoft.com/en-us/configmgr/mdt/)

# Chocolatey

You can use the free version for this, but using the Business edition is preferable, as you can take apps installed outside of Chocolatey back under the choco management and therefore patch them - this is a really great feature of Chocolatey in my opinion.

## Scenario

To set the scene, this is the environment the Chocolatey patching will work well in:

* As we're using the free version of Chocolatey in this example, you will have already setup many apps to be installed through Chocolatey, publishing them as **available** in the Company Portal using Intune.
* Apps are made **available** and use a Chocolatey PowerShell script to always install the **latest version** - this means the detection method (custom script detection) will still work, even if the apps are updated and the deployment is set to **required**.
* These apps will now be out of date as they are not automatically updating and/or the users don't have local admin rights if they are installed in the system context.

## Intune app install scripts

An example of PowerShell scripts I used to install apps using Chocoately, deployed through Intune.
Note: These are *not* the scripts used for patching the existing apps:

* [**Install** latest version of Adobe Reader DC (with auto-update disabled) from Chocolatey:](https://github.com/jrudlin/Intune/blob/master/Install-AdobeReaderDC-Choco.ps1)
* [**Uninstall** Adobe Reader DC using Chocolatey:](https://github.com/jrudlin/Intune/blob/master/Uninstall-AdobeReaderDC-Choco.ps1)
* [Intune **custom detection** method for Adobe Reader DC installed through Chocolatey](https://github.com/jrudlin/Intune/blob/master/Detection-Rule-AdobeReaderDC-Choco.ps1)

# Getting Started

So now you want to patch/update all of your apps using Chocolatey, through Intune.

## PSADT

Grab a copy of the latest [PSADT](https://psappdeploytoolkit.com) and lets start to edit the **Deploy-Application.ps1**, or you can grab my [ready made copy](https://github.com/jrudlin/Intune/blob/master/Choco%20App%20Patching/PSADT/Deploy-Application.ps1) and skip this PSADT section.

### PRE-INSTALLATION

In the PRE-INSTALLATION section, we'll add a check to see if Chocolatey is installed and install if it's not. At the time, Win32 dependencies weren't available in Intune.

```powershell
##*===============================================
##* PRE-INSTALLATION
##*===============================================
[string]$installPhase = 'Pre-Installation'
# Check if Chocolatey is installed and if not, install it.
If ( get-command -Name choco.exe -ErrorAction SilentlyContinue ){
  $ChocoExe = "choco.exe"
} elseif (test-path -Path (Join-Path -path $Env:ALLUSERSPROFILE -ChildPath "Chocolatey\bin\choco.exe")) {
  $ChocoExe = Join-Path -path $Env:ALLUSERSPROFILE -ChildPath "Chocolatey\bin\choco.exe"
} else {
  try {
    Write-Log -Message "Starting to install chocolatey...." -Severity 1 -Source $deployAppScriptFriendlyName
    Invoke-Expression ((New-Object -TypeName net.webclient).DownloadString('https://chocolatey.org/install.ps1')) -ErrorAction Stop
    choco feature enable -n allowGlobalConfirmation
    $ChocoExe = Join-Path -path $Env:ALLUSERSPROFILE -ChildPath "Chocolatey\bin\choco.exe"
  }
  catch {
    Write-Log -Message "Failed to install Chocolatey" -Severity 3 -Source $deployAppScriptFriendlyName
    Throw "Failed to install Chocolatey"
  } 
}
```

Now for the key component, the check for the outdated apps. Choco will scan the existing installations (in the free version, only the apps that were previously installed with Choco).

Once we have a list of outdated applications that can be updated through choco, throw a default PSADT popup with a pre-populated list of common apps that should be closed:

![ChocoPatching1]({{ site.baseurl }}/assets/images/ChocoAppPatching1.jpg)

Optionally allow the user to defer:

![ChocoPatching2]({{ site.baseurl }}/assets/images/ChocoAppPatching2.jpg)

and then the list of apps to be updated:

![ChocoPatching3]({{ site.baseurl }}/assets/images/ChocoAppPatching3.jpg)

```powershell
# Check if any packages actually need updating
$OutdatedList = & $ChocoExe outdated --limit-output
If(($OutdatedList -gt 0)-and(-not($OutdatedList -like "*Error*"))){

  Write-Log -Message "Chocolatey packages needing updates: [$($OutdatedList.Count)]" -Severity 1 -Source $deployAppScriptFriendlyName

  ## Show Welcome Message, close Internet Explorer if required, allow up to 3 deferrals, verify there is enough disk space to complete the install, and persist the prompt
  Show-InstallationWelcome -CloseApps 'Outlook=Microsoft Outlook,winword=Microsoft Office Word,excel=Microsoft Office Excel,VISIO=Visio,ONENOTE=OneNote,POWERPNT=Microsoft PowerPoint,MSPUB=Microsoft Publisher,AcroRd32=Adobe Reader' -AllowDefer -DeferTimes 3 -CheckDiskSpace -PersistPrompt -BlockExecution
  # 'Outlook=Microsoft Outlook,winword=Microsoft Office Word,excel=Microsoft Office Excel,VISIO=Visio,ONENOTE=OneNote,POWERPNT=Microsoft PowerPoint,MSPUB=Microsoft Publisher,AcroRd32=Adobe Reader'
  ## Show Progress Message (with the default message)
    
            $ListFormatted = $OutdatedList | % { `
                    [pscustomobject]@{ `
                        Name = ($_ -split '\|')[0]; `
                        NewVersion = ($_ -split '\|')[2] `
                    } `
            }
            Show-InstallationProgress -StatusMessage "Installation in progress...`n$($ListFormatted | % {"`n$($_.Name), $($_.NewVersion)"})"
```

### INSTALLATION

This code is actually a continuation of the above, so you'll need the whole lot otherwise the braces {} won't equate.

Here we are initiating the upgrade of all the outdated apps and finally displaying a message with either the succesfully installed or app that failed to upgrade:

![ChocoPatching4]({{ site.baseurl }}/assets/images/ChocoAppPatching4.jpg)

```powershell
## <Perform Installation tasks here>
If ($ChocoExe){
  Write-Log -Message "Starting Choco installs..." -Severity 1 -Source $deployAppScriptFriendlyName		
  Execute-Process -Path $ChocoExe -Parameters 'upgrade all --confirm'
  
          #Show-InstallationProgress -StatusMessage "Some apps attempted install..."

  start-sleep 5
  $FailedList = & $ChocoExe outdated --limit-output
  Write-Log -Message "Failed apps: [$($FailedList.Count)]" -Severity 1 -Source $deployAppScriptFriendlyName		
  If($FailedList -gt 0){
    
              $FailedListFormatted = $FailedList | % { `
                  [pscustomobject]@{ `
                      Name = ($_ -split '\|')[0]; `
                      CurrentVersion = ($_ -split '\|')[1]; `
                  } `
              }
              Show-InstallationPrompt -Message "The follow apps could not be updated: `n$($FailedListFormatted | % {"`n$($_.Name), $($_.CurrentVersion)"}) `n`nThe installation will retry again later." -ButtonRightText 'OK' -Icon Warning -NoWait
  
          } else {
              Show-InstallationPrompt -Message "Apps succesfully updated: `n$($ListFormatted | % {"`n$($_.Name), $($_.NewVersion)"})" -ButtonRightText 'OK' -Icon Information -NoWait
  }

}

} else {

  Write-Log -Message "No Chocolatey packages determined as needing to be updated" -Severity 2 -Source $deployAppScriptFriendlyName

}
```

## Intune

There are a few moving parts to this so lets have an explanation of the remaining files (that can be found in the download link at the bottom of this post).

 * Install.cmd - calls the Deploy-Application.exe using the ServiceUI.exe (Part of the Microsoft Deployment Toolkit (MDT)). This should be in the same folder as the PSADT.
 * ServiceUI.exe - called by above Install.cmd
 * ChocoPackageUpdates-ScheduledTask.ps1 - creates a local scheduled task to run monthly or quaterly, and copies the PSADT folder contents locally. The task calls the Install.cmd when it's run.
 * Detection-Rule-Monthly.ps1 and Detection-Rule-Quarterly.ps1 - Don't deploy these to the endpoints as they are the scripts that I used for the detection method in Intune - they detect if the Scheduled Task exists.



### Deployment

In this solution, we have one Azure AD Group for **monthly updates** (IT/Dev/Test machines), and another for **quarterly updates** (Rest of the business) - you can of course expand this to be more granular using more Scheduled Tasks.

The app contents is detailed above. Create it with your **IntuneWinAppUtil.exe** wrapper.

Deploy the monthly servicing app to the Azure AD group with your pilot machines using install cmdline:
```powershell
powershell.exe ChocoPackageUpdates-ScheduledTask.ps1 -Release Monthly
```

And the quarterly servicing:

```powershell
powershell.exe ChocoPackageUpdates-ScheduledTask.ps1 -Release Quarterly
```

## Scripts Download

All the files/scripts used are available for [download](https://github.com/jrudlin/Intune/tree/master/Choco%20App%20Patching) on my Github repo.