---
layout: post
title: PowerShell WMI function to retrieve .cab file UNC content location from SCCM
  Deployment Package
date: 2018-07-18 20:23:58.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- PowerShell
- SCCM
- Windows 10
tags:
- offline servicing
- SCCM deployment package update cab file location
- SCCM software updates
- SCCM WMI query
- WIM image maintenance
- windows 10 image servicing
meta:
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '20150780867'
  timeline_notification: '1531945439'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1019679195826409473";}}
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  publicize_linkedin_url: www.linkedin.com/updates?topic=6425444890548797440
  _publicize_done_18837845: '1'
  _wpas_done_18662064: '1'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/07/18/powershell-wmi-function-to-retrieve-cab-file-unc-content-location-from-sccm-deployment-package/"
---
There are lots of great scripts out there for performing offline Windows 10 WIM image servicing using DISM and PowerShell, like [@jarwidmark](https://twitter.com/jarwidmark)'s [Create-W10RefImageViaDISM.ps1](https://github.com/DeploymentResearch/DRFiles/blob/master/Scripts/Create-W10RefImageViaDISM.ps1) script that is on GitHub.

But......all of these require that you separately download the .msu (windows updates) from the Microsoft catalog prior to running the servicing script.

This is another manual step and seems unnecessary to me, when we know the updates have most likely already been downloaded by SCCM.

Therefore I wrote a small **function to retrieve the .cab files' UNC location** from the SCCM deployment package.(yep, it doesn't have to be the .msu for servicing, it can be the equivalent .cab file as well)

You will need access to the WMI (SMS) provider on the SCCM site server for this to work.  
Also, SCCM must have already downloaded the updates into a deployment package stored on a file share. I hope you have this part automated using an ADR already.

1. First specify your **CAS or Primary Site server name** where the SMS Provider is running so that you can query WMI:
```powershell
# Specific SCCM site details  
$siteserver = "SCCMCAS1.domain.local"  
$sitecode = "CAS"  
```

2. The WMI class we will use is the **SMS_SoftwareUpdate** amongst others, but this is the most important and regularly used one, so I've set it as a variable.  
The namespace is built from the **$sitecode** above.
```powershell
$class = "SMS_SoftwareUpdate"  
$NameSpace = "root\SMS\Site\$sitecode"  
```

3. The most important part, the WMI filter ($UpdatesFilter) used to pick specific updates based on your criteria from the SCCM Software Updates database. In this example I wanted only Windows 10 1803 x64 updates with the matching articleID.
```powershell
$UpdatesFilter = "LocalizedDisplayName like '%1803 for x64%' and articleid="  
```

4. As explained by [@jarwidmark](https://twitter.com/jarwidmark) and [@miketerrill](https://twitter.com/miketerrill), a servicing stack update, an Adobe Flash update and the monthly Cumulative Update's are all required when servicing an offline Windows 10 WIM, so specify the ID's of these in the variables. (Note, you need to search online for these as Microsoft does not maintain a list of the latest Servicing Stack updates).
```powershell
$ServicingUpdateID = "4343669"  
$AdobeFlashUpdateID = "4338832"  
$MonthlyCUID = "4338819"  
```

5. The function that basically pieces together the path of the .cab file that was downloaded into your SCCM Deployment Package (could have been via an ADR or directly from the update/SUG).
From the update/deployment package in WMI, we can tell the top level folder of where the package contents will be. This is **$UpdatePackage.PkgSourcePath.**  
The folder inside the PkgSourcePath is a unique guid. This comes from the **ContentID** of the update itself.  
Finally the filename is easy to get. We pull the update from **SMS_CIContentFiles** and get the **.FileName** property.
**Note:** The function should really be changed to pass the **$UpdatesFilter** as well. This would be very easy to do.

```powershell
Function Get-SoftwareUpdatePath {
param (
[string]$UpdateArticleID
)
$UpdatesFilterArticleID = "$UpdatesFilter$UpdateArticleID"
 
$Update = get-wmiobject -ComputerName $siteserver -Query "select * from $class where $UpdatesFilterArticleID" -Namespace $NameSpace
$UpdateCIID = $Update.CI_ID
If($UpdateCIID.count -gt 1){Write-Warning "Please check UpdatesFilter variable as more than one update was returned"; Break}
 
$UpdateContent = get-wmiobject -ComputerName $siteserver -Query "select * from SMS_CItoContent where ci_id=$UpdateCIID" -Namespace $NameSpace
$UpdateContentID = $UpdateContent.ContentID
$UpdateFolder = $UpdateContent.ContentUniqueID
 
$objContent = Get-WmiObject -ComputerName $siteserver -Namespace $NameSpace -Class SMS_CIContentFiles -Filter "ContentID = $UpdateContentID"
$UpdateFileName = $objContent.FileName
 
$UpdatePackage = Get-WmiObject -Namespace root/SMS/site_$($SiteCode) -ComputerName $SiteServer -Query "SELECT DISTINCT sup.* FROM SMS_SoftwareUpdatesPackage AS sup `
JOIN SMS_PackageToContent AS pc ON sup.PackageID=pc.PackageID JOIN SMS_CIToContent AS cc ON pc.ContentID = cc.ContentID WHERE CC.CI_ID='$UpdateCIID'"
If($UpdatePackage.Count -gt 1){
$UpdatePackage = $UpdatePackage[0]
} elseif($UpdatePackage.Count -eq ""){
Write-Error "Could not find Software Deployment Package for update: $($update.LocalizedDisplayName)"
}
 
$UpdateFullPath = "$($UpdatePackage.PkgSourcePath)\$UpdateFolder\$UpdateFileName"
 
return $UpdateFullPath
}  
```

6. Run each line to return the UNC path of the .cab file.
```powershell
Get-SoftwareUpdatePath -UpdateArticleID $ServicingUpdateID  
Get-SoftwareUpdatePath -UpdateArticleID $AdobeFlashUpdateID  
Get-SoftwareUpdatePath -UpdateArticleID $MonthlyCUID  
```

7. The output should look something like:
```powershell
\\domain.local\packages\softwareupdates\win10\4668f609-e85c-4384-a758-6251918f1246\Windows10-KB4343669-x64.cab  
\\domain.local\packages\softwareupdates\win10\edc45df9-43c7-4a48-8035-72557d2b598d\Windows10-KB4338832-x64.cab  
\\domain.local\packages\softwareupdates\win10\07d1a4be-d7ab-4b9d-be49-19fe20b81754\Windows10-KB4338819-x64.cab  
```

You can now pass these .cab files (as in the PoSh varables) to your DISM/PowerShell scripts for servicing the Windows 10 WIM file, further automating your way to an easy life :) and saving you from having to download the .msu files manually.

