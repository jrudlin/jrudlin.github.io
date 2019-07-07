---
layout: post
title: Download all Office 2016 updates with PowerShell and SCCM
date: 2017-08-12 16:41:31.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- PowerShell
- SCCM
tags:
- Office 2016
meta:
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '8198069943'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2017/08/12/download-all-office-2016-updates-with-powershell-and-sccm/"
---
Hi Guys,

Sorry for the delay, been working on a big project to rollout thin clients across libraries in a London borough. I will be writing up a collection posts which will make up the automation of a cloud based VDI solution in Azure, starting off today with keeping the Office 2016 MSI/Volume License edition app up to date in SCCM.

When deploying any modern Office version with SCCM, you want to keep it up to date by adding all the latest .msp files to the /updates folder.

I adapted some popular scripts found online ┬áand added what I thought were cool additions.

This one:┬á[maintaining-your-office-2016-installation-source](http://www.nowmicro.com/blog/maintaining-your-office-2016-installation-source)┬áinvolved creating a new SUG and SU Package in SCCM, a bit yucky.

This one:┬á[download-office-2016-proplus-x86-updates](http://fredbainbridge.com/2017/01/11/download-office-2016-proplus-x86-updates/)┬áhad a static list that has to be maintained by the script owner and used custom guids to name the files.

Both these issues are 'resolved' using the following code.  
Pre-reqs:

- 
  - Access to the ConfigManager PSD1 PoSh module. You can load the module in a Admin PoSh prompt like:

[code language="powershell"]  
 $ConfigMgr\_psd1\_path = $Env:SMS\_ADMIN\_UI\_PATH.Substring(0,$Env:SMS\_ADMIN\_UI\_PATH.Length-5) + '\ConfigurationManager.psd1'

Import-Module $ConfigMgr\_psd1\_path  
[/code]

- SCCM Sofware Updates sync configured to download Office 2016 metadata
- Full Administrator access to SCCM Primary Site Server

[code language="powershell"]  
$siteserver = "srv-sccm.domain.local"  
$sitecode = "AB0"  
$class = "SMS\_SoftwareUpdate"  
$NameSpace = "root\SMS\Site\_$sitecode"  
$StagingLocation = "T:\O2016\_Updates"  
$MSPsFolder = "$StagingLocation\ready"\</code\>

Function ConvertFrom-Cab  
{  
 [CmdletBinding()]  
 Param(  
 $cab,  
 $destination,  
 $filename  
 )  
 $comObject = "Shell.Application"  
 Write-Verbose "Creating $comObject"  
 $shell = New-Object -Comobject $comObject  
 if(!$?) { $(Throw "unable to create $comObject object")}  
 Write-Verbose "Creating source cab object for $cab"  
 $sourceCab = $shell.Namespace("$cab").items()  
 if(-not (Test-Path $destination))  
 {  
 Write-Verbose "Creating destination folder object for $destination"  
 new-item $destination -ItemType Directory  
 }  
 $DestinationFolder = $shell.Namespace($destination)  
 Write-Verbose "Expanding $cab to $destination"  
 $DestinationFolder.CopyHere($sourceCab)  
 ForEach($Cabitem in $sourceCab){  
 If($Cabitem.name -like "\*.msp"){  
 Rename-Item -Path "$destination\$($Cabitem.name)" -NewName ($fileName)  
 }  
 }  
}

Remove-Item $StagingLocation -Force -Confirm:$False -Recurse  
$Office2016Updates = Get-CMSoftwareUpdate -Name "\*Microsoft\*2016\*32-Bit\*" -Fast | ? {($\_.IsExpired -ne "False") -and ($\_.IsSuperseded -ne "False")}  
md $StagingLocation

ForEach($Update in $Office2016Updates){  
 $CI\_ID = $Update.CI\_ID  
 $ContentID = (get-wmiobject -ComputerName $siteserver -Query "select \* from SMS\_CItoContent where ci\_id=$CI\_ID" -Namespace $NameSpace).ContentID  
 $objContent = Get-WmiObject -ComputerName $siteserver -Namespace $NameSpace -Class SMS\_CIContentFiles -Filter "ContentID = $ContentID"  
 $Filename1 = "KB$((([uri]$Update.LocalizedInformativeURL).AbsolutePath -split "/")[2])"  
 $CabFileName = $objContent.Filename  
 $FileName2 = ($CabFileName -split ".cab")[0]  
 $FileNameWithoutExtension = $FileName2 + "\_" + $Filename1  
 write-host "FileNameWithoutExtension will be: $FileNameWithoutExtension"  
 $URL = $objContent.SourceURL  
 write-host "$URL" -ForegroundColor Green  
 try  
 {  
 $FileNamePath = "$StagingLocation\$CabFileName"  
 write-host "FileNamePath = $FileNamePath"  
 Start-BitsTransfer -Source $URL -Destination $FileNamePath  
 If(Test-Path $FileNamePath)  
 {  
 Write-Output "$FileNamePath found, extracting files from cab..."  
 ConvertFrom-Cab -cab $FileNamePath -destination $MSPsFolder -filename $($FileNameWithoutExtension+".msp")  
 write-host "Deleting items cabs from $StagingLocation"  
 Get-ChildItem -Path $StagingLocation -File -Filter \*.cab | Remove-Item -Force -Confirm:$false  
 write-host "Deleting items xmls from $MSPsFolder"  
 Get-ChildItem -Path $MSPsFolder -File -Filter \*.xml | Remove-Item -Force -Confirm:$false  
 }  
 }  
 catch  
 {  
 write-host "stopping here"  
 }  
}

[/code]

In your output folder $MSPsFolder you will have all the MSPs you need to copy into the /updates folder in the Office 2016 app/package source.

