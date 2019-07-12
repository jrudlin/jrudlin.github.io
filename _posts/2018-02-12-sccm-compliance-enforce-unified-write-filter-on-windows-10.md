---
layout: post
title: SCCM Compliance - enforce Unified Write Filter on Windows 10
date: 2018-02-12 14:42:30.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- PowerShell
- SCCM
tags:
- Compliance
- DCM
- Deep Freeze
- Unified Write Filter
- UWF
- Windows 10
meta:
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:56:"https://twitter.com/JackRudlin/status/963060756139925504";}}
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '14668943831'
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  timeline_notification: '1518446552'
  _publicize_failed_18837845: O:13:"Keyring_Error":2:{s:6:"errors";a:1:{s:21:"keyring-request-error";a:1:{i:0;a:6:{s:7:"headers";O:42:"Requests_Utility_CaseInsensitiveDictionary":1:{s:7:"
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/02/12/sccm-compliance-enforce-unified-write-filter-on-windows-10/"
---
I am in the final stages of deploying a new solution to libraries across a borough in London. The solution involved replacing a Windows 7 physical desktop that had the Deep Freeze product, with Windows 10.

I didn't want to use Deep Freeze as the console is clunky and it's yet another agent to manage on the client OS. Windows 10 has the Unified Write Filter (UWF) built-in and this would do the job perfectly.

All the desktops are managed using SCCM and SCCM fully supports the Windows write filters.

I started off looking at basic batch files calling uwfmgr.exe, but things quickly got messy, so I switched to trusty old WMI, but managed using PowerShell and SCCM of course :)

I've tried to break it down into sections.

It is best practice to have the pagefile on a separate disk when using UWF as it cannot be added to the exclusions list, so here we have a function that we can pass in your new volume and remove the NTFS permissions to stop normal users from fiddling:

[code language="powershell"]

function Remove-NTFSPermissions($folderPath, $accountToRemove, $permissionToRemove) {

$fileSystemRights = [System.Security.AccessControl.FileSystemRights]$permissionToRemove

$inheritanceFlag = [System.Security.AccessControl.InheritanceFlags]"ContainerInherit, ObjectInherit"

$propagationFlag = [System.Security.AccessControl.PropagationFlags]"None"

$accessControlType =[System.Security.AccessControl.AccessControlType]::Allow

$ntAccount = New-Object System.Security.Principal.NTAccount($accountToRemove)

if($ntAccount.IsValidTargetType([Security.Principal.SecurityIdentifier])) {

$FileSystemAccessRule = New-Object System.Security.AccessControl.FileSystemAccessRule($ntAccount, $fileSystemRights, $inheritanceFlag, $propagationFlag, $accessControlType)

$oFS = New-Object IO.DirectoryInfo($folderPath)

$DirectorySecurity = $oFS.GetAccessControl([System.Security.AccessControl.AccessControlSections]::Access)

$DirectorySecurity.RemoveAccessRuleAll($FileSystemAccessRule)

$oFS.SetAccessControl($DirectorySecurity)

return "Permissions " + $permissionToRemove + " Removed on " + $folderPath + " folder"

}

return 0

}

[/code]

The main function to enable the Unified Write Filter, configure the overlay, exclusions and pagefile.

First part of the function is to check if UWF is currently enabled or not. If we need to enable it, it will require a restart before the write filter becomes active.

[code language="powershell"]  
function Enable-UWF {

# Global variables  
 $NAMESPACE = "root\standardcimv2\embedded"

########################################################################################################################

# Get current state of UWF  
 $objUWFFilter = Get-WmiObject -Namespace $NAMESPACE -Class UWF\_Filter;

if(!$objUWFFilter) {  
 write-output "`nUnable to retrieve Unified Write Filter settings. from $NAMESPACE" | out-file -FilePath $env:temp\UWF.log -Append  
 return;  
 }

# Check if UWF is enabled  
 if(($objUWFFilter.CurrentEnabled)-or($objUWFFilter.NextEnabled)) {  
 write-output "`nUWF Filter is enabled" | out-file -FilePath $env:temp\UWF.log -Append  
 } else {  
 write-output "`nUWF Filter is NOT enabled, enabling now..." | out-file -FilePath $env:temp\UWF.log -Append

# Call the method to enable UWF after the next restart. This sets the NextEnabled property to false.  
 $retval = $objUWFFilter.Enable();

# Check the return value to verify that the enable is successful  
 if ($retval.ReturnValue -eq 0) {  
 write-output "Unified Write Filter will be enabled after the next system restart." | out-file -FilePath $env:temp\UWF.log -Append  
 } else {  
 "Unknown Error: " + "{0:x0}" -f $retval.ReturnValue  
 }

}  
[/code]

Once the computer has restarted, SCCM compliance will run again at your specified evaluation schedule, at which point the following (also still part of the Enable-UWF function) will run to protect the C drive:

[code language="powershell"]  
 # Only perform config if after the next restart the UWF is enabled  
 $objUWFFilter = Get-WmiObject -Namespace $NAMESPACE -Class UWF\_Filter;  
 If($objUWFFilter.NextEnabled){  
 write-output "UWF is set to enabled after next restart, continue to check config" | out-file -FilePath $env:temp\UWF.log -Append

# Get volume protect state  
 $objUWFVolumeC = Get-WmiObject -Namespace $NAMESPACE -Class UWF\_Volume -Filter "CurrentSession = false" | ? {(get-volume -DriveLetter C).UniqueId -like "\*$($\_.VolumeName)\*"}

# Check if C is protected  
 If(!$objUWFVolumeC.Protected){  
 write-output "C Drive not protected, will enable protection now.." | out-file -FilePath $env:temp\UWF.log -Append  
 #enable protection  
 #$retval = $objUWFVolumeC.Protect()  
 uwfmgr.exe volume protect c:  
 # Check the return value to verify that it was successful  
 #if ($retval.ReturnValue -eq 0) {  
 # write-host "Unified Write Filter will protect the C drive after the next system restart." -ForegroundColor Green  
 #} else {  
 # "Unknown Error: " + "{0:x0}" -f $retval.ReturnValue  
 #}  
 }  
[/code]

Set the overlay size based on the % of free space remaining on the disk:  
[code language="powershell"]  
 # Overlay size and type

$objUWFOverlayConfig = Get-WmiObject -Namespace $NAMESPACE -Class UWF\_OverlayConfig -Filter "CurrentSession = false"

If($objUWFOverlayConfig.MaximumSize -le 1024){  
 # need to set maximum size  
 $OverlaySize = (Get-Volume -DriveLetter C).SizeRemaining-((Get-Volume -DriveLetter C).SizeRemaining/2) | % {[math]::truncate($\_ /1MB)}  
 write-output "`nTry to set overlay max size to $OverlaySize MB." | out-file -FilePath $env:temp\UWF.log -Append  
 $objUWFOverlayConfig.SetMaximumSize($OverlaySize);  
 $WarningSize = [math]::Round($OverlaySize/10\*8)  
 $CriticalSize = [math]::Round($OverlaySize/10\*9)  
 uwfmgr.exe overlay set-warningthreshold $WarningSize  
 uwfmgr.exe overlay set-criticalthreshold $CriticalSize  
 }

If($objUWFOverlayConfig.Type -ne 1){  
 # Set overlay type to Disk based  
 write-host "`nTry to set overlay type to Disk based" -ForegroundColor Yellow  
 $objUWFOverlayConfig.SetType(1)  
 }  
[/code]

Set all the file and registry exclusions based on best practice and Microsoft recommendations (links included in the code below):  
[code language="powershell"]  
# File exclusions

$objUWFVolumeC = Get-WmiObject -Namespace $NAMESPACE -Class UWF\_Volume -Filter "CurrentSession = false" | ? {(get-volume -DriveLetter C).UniqueId -like "\*$($\_.VolumeName)\*"}  
 $FileExclusionList = @(

#Exclusions for Defender and SCEP https://msdn.microsoft.com/en-us/library/windows/hardware/mt571986(v=vs.85).aspx  
 "\ProgramData\Microsoft\Microsoft Security Client", `  
 "\ProgramData\Microsoft\Windows Defender", `  
 "\Program Files\Windows Defender", `  
 "\Program Files (x86)\Windows Defender", `  
 "\Users\All Users\Microsoft\Microsoft Security Client", `  
 "\Windows\WindowsUpdate.log", `  
 "\Windows\Temp\MpCmdRun.log", `

#BITS: https://msdn.microsoft.com/en-us/library/windows/hardware/mt571989(v=vs.85).aspx  
 "\Users\All Users\Microsoft\Network\Downloader", `

#https://docs.microsoft.com/en-us/windows-hardware/customize/enterprise/uwfexclusions  
 "\Windows\wlansvc\Policies", `  
 "\Windows\dot2svc\Policies", `  
 "\ProgramData\Microsoft\wlansvc\Profiles\Interfaces", `  
 "\ProgramData\Microsoft\dot3svc\Profiles\Interfaces"

)

write-host "`n"

ForEach($File in $FileExclusionList){

If(!($objUWFVolumeC.FindExclusion($File)).bFound){

write-host "$File needs to be added to exclusions"  
 $objUWFVolumeC.AddExclusion($File)

}

}

# Reg exclusions  
 $objUWFRegFilter = Get-WmiObject -Namespace $NAMESPACE -Class UWF\_RegistryFilter -Filter "CurrentSession = false"

$RegExclusionList = @(

#Exclusions for Defender and SCEP https://msdn.microsoft.com/en-us/library/windows/hardware/mt571986(v=vs.85).aspx  
 "HKLM\SOFTWARE\Microsoft\Windows Defender", `  
 "HKLM\SOFTWARE\Microsoft\Microsoft Antimalware",

#https://docs.microsoft.com/en-us/windows-hardware/customize/enterprise/uwfexclusions  
 "HKLM\Software\Microsoft\Windows\CurrentVersion\BITS\StateIndex", `  
 "HKLM\SOFTWARE\Policies\Microsoft\Windows\Wireless\GPTWirelessPolicy", `  
 "HKLM\SOFTWARE\Policies\Microsoft\Windows\WiredL2\GP\_Policy", `  
 "HKLM\SYSTEM\CurrentControlSet\services\Wlansvc", `  
 "HKLM\SYSTEM\CurrentControlSet\services\WwanSvc", `  
 "HKLM\SYSTEM\CurrentControlSet\services\dot3svc", `  
 "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Time Zones", `  
 "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation"

)

ForEach($Reg in $RegExclusionList){

If(!($objUWFRegFilter.FindExclusion($Reg)).bFound){

write-host "$Reg needs to be added to exclusions"  
 $objUWFRegFilter.AddExclusion($Reg)

}

}

} else {

write-host "UWF is not set to enabled on the next restart, will not check config"

}  
[/code]

Create a new volume off of the free space on the disk which will will be used to host the pagefile. Once the volume is created, set the PageFileSetting to use this new drive:  
[code language="powershell"]  
 # Pagefile creation on a separate volume  
 $PageFileDriveLetter = "P"  
 $PageFileDriveSizeGB = 5  
 # Check page file does not exist  
 $PFUsage = Get-WmiObject -Class Win32\_PageFileUsage -Property Name  
 If(!($PFUsage) -or ($($PFUsage.Name) -eq "C:\pagefile.sys")){  
 Write-Warning "Pagefile does not exist, will create one on a $PageFileDriveLetter drive"

# create page file drive if does not exist  
 $PageFileDrive = Get-CimInstance -ClassName CIM\_StorageVolume -Filter "Name='$($PageFileDriveLetter):\\'"  
 If(!$PageFileDrive){

Write-Warning -Message "Failed to find the DriveLetter $PageFileDriveLetter specified, creating new volume now...."  
 $CVol = get-volume -DriveLetter C  
 $VolSizeRemaining = [int]($CVol.SizeRemaining /1GB).ToString(".")  
 If($VolSizeRemaining -lt $PageFileDriveSizeGB){  
 Write-Error "Not enough free space on the C drive to create a new volume for the page file"  
 return  
 } else {  
 write-host "Enough free space on the C drive is available to create the new $PageFileDriveLetter drive"  
 #enable optimise drives service (defrag) otherwise resize fails  
 Set-Service -Name defragsvc -StartupType Manual -ErrorAction SilentlyContinue

$NewCDriveSize = $CVol.Size-"$($PageFileDriveSizeGB)GB"  
 write-host "Resizing C: from $($CVol.Size) to $NewCDriveSize"  
 Get-Partition -DriveLetter C | Resize-Partition -Size $NewCDriveSize -ErrorAction Stop  
 write-host "Resized C to $NewCDriveSize. Now creating new $PageFileDriveLetter drive from the free space..."  
 # Create new partition  
 Get-Volume -DriveLetter C | Get-Partition | Get-Disk | New-Partition -UseMaximumSize -DriveLetter $PageFileDriveLetter | Format-Volume

}

} else {  
 write-host "$PageFileDriveLetter already exists"  
 }

write-host "Creating page file on $PageFileDriveLetter drive"  
 New-CimInstance -ClassName Win32\_PageFileSetting -Property @{Name= "$($PageFileDriveLetter):\pagefile.sys"} -ErrorAction Stop | Out-Null  
 $InitialSize = [math]::Round((get-volume -DriveLetter $PageFileDriveLetter).SizeRemaining /1MB /10 \*9)  
 $MaximumSize = [math]::Round((get-volume -DriveLetter $PageFileDriveLetter).SizeRemaining /1MB /10 \*9)  
 # http://msdn.microsoft.com/en-us/library/windows/desktop/aa394245%28v=vs.85%29.aspx  
 Get-CimInstance -ClassName Win32\_PageFileSetting -Filter "SettingID='pagefile.sys @ $($PageFileDriveLetter):'" -ErrorAction Stop | Set-CimInstance -Property @{  
 InitialSize = $InitialSize ;  
 MaximumSize = $MaximumSize ;  
 } -ErrorAction Stop

Write-Verbose -Message "Successfully configured the pagefile on drive letter $DriveLetter"

} else {

write-host "Pagefile already exists: $($PFUsage.Name)"

}  
[/code]

Bringing it all together, calling the functions above, and installing the UWF feature if its not currently:  
[code language="powershell"]  
$UWF\_Feature = (Get-WindowsOptionalFeature -Online -FeatureName Client-UnifiedWriteFilter -ErrorAction SilentlyContinue).State

If($UWF\_Feature -eq "Disabled"){

write-host "Not installed"  
 Enable-WindowsOptionalFeature -Online -FeatureName Client-UnifiedWriteFilter -All -ErrorAction SilentlyContinue

write-host "Please run this script again after a restart, to enable UWF filter" -ForegroundColor Yellow

pause

exit 3010

} else {

write-host "`nClient-UnifiedWriteFilter WindowsOptionalFeature installed, now configure UWF" -ForegroundColor Green

Enable-UWF -ErrorAction SilentlyContinue

If(test-path p:){  
 $folder = "P:\"  
 Remove-NTFSPermissions $folder "Authenticated Users" "Modify"  
 Remove-NTFSPermissions $folder "Users" "ReadAndExecute"  
 }

}  
[/code]

