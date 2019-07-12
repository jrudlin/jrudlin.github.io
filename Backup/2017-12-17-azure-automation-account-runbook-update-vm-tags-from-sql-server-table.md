---
layout: post
title: Azure Automation Account Runbook - Update VM tags from SQL Server table
date: 2017-12-17 20:47:04.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Automation
- Azure
- PowerShell
- SQL
tags:
- Azure Automation
- SQL Server
- VM Tags
meta:
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:56:"https://twitter.com/JackRudlin/status/942496393985478656";}}
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '12618361394'
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=13673999&stype=M&topic=6348262087101415424&type=U&a=W4uG
  _publicize_done_18837845: '1'
  _wpas_done_18662064: '1'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2017/12/17/azure-automation-account-runbook-update-vm-tags-from-sql-server-table/"
---
As you can probably see from previous posts, I spend a lot more time these days automating tasks with PowerShell using Azure Automation hybrid workers.

In this post, I wanted to show you a way of reading from a SQL Server database the opening hours of a remote site/office and inputting these times into Azure VMs tags.

This ensures that Azure VMs are available during office hours and also helps to keeps Azure running costs down.

[code language="powershell"]

# Get office opening times from SQL database

param  
 (  
 [Parameter(Mandatory = $false)]  
 [string] $SQLServer = "SQL1.test.local",

[Parameter(Mandatory = $false)]  
 [string] $Database = "Sites",

[Parameter(Mandatory = $false)]  
 [string] $RGbaseName = "RG-WIN10-",

[Parameter(Mandatory = $false)]  
 [string] $AzureSubscriptionName = "azuresub",

[Parameter(Mandatory = $false)]  
 [string] $Filter = "\*WIN7\*"

)

$ErrorActionPreference = "Stop"

[/code]

Next is a simple function that calls a separate runbook that sends an email report on all the output of the last job run of the specified runbook (let me know if you want more details on this):

[code language="powershell"]

Function Send-EmailReport  
{  
 #Send email report of this runbook job  
 $params = @{AutomationRunBookName="OpeningTimesToTags";SendReportByEmail=$true;SendToAddress="ITSupport@domain.gov.uk","ITSupport2@domain.gov.uk"}  
 Start-AzureRmAutomationRunbook -AutomationAccountName "AA-WE-SUB-01" -Name "Report-AutomationJobOutput" -ResourceGroupName "RG-AA-01" ÔÇôParameters $params -RunOn "AA-HybridWorker" \>$null 2\>&1 | out-null  
}

[/code]

Grab the credentials needed to run the remainder of the script from the AA credentials vault:

[code language="powershell"]  
# Azure Global Admin  
$AzureOrgIdCredential = Get-AutomationPSCredential -Name 'sv-azure-automation'

# Database Read Only account  
$DB\_RO = Get-AutomationPSCredential -Name 'DB Read Only'

# Login to Azure  
Login-AzureRmAccount -Credential $AzureOrgIdCredential \>$null 2\>&1 | out-null

#Select the specified subscription now  
Select-AzureRMSubscription -SubscriptionName $AzureSubscriptionName \>$null 2\>&1 | out-null  
[/code]

Load the SQLServer PowerShell module or install if not currently. This one is great as we no longer need SSMS:

[code language="powershell"]  
# SqlServer PoSh Module  
If(-not(get-module -ListAvailable -Name SqlServer)){  
 write-output "Could not find module 'SqlServer', installing now..."  
 Find-Module SqlServer | Install-Module  
}

import-module -Name SqlServer -ErrorAction SilentlyContinue

If(Get-Module -Name SqlServer){  
 write-output "Module 'SqlServer' loaded, continue"  
} else {  
 Send-EmailReport  
 write-error "Could not load or install 'SqlServer' PoSh Module"  
}  
[/code]

Get all the VMs from Azure where the Resource Group name is like $RGbaseName:  
[code language="powershell"]  
write-output "`nLooping through all RGs like $RGbaseName to find specific VMs"  
$RGs = Get-AzureRmResourceGroup | ? ResourceGroupName -like "$RGbaseName\*"  
If(!$RGs){write-error "Could not find any Resource Groups with a name like $RGbaseName\*";return} else  
{  
 write-output "Retrieved $($RGs.count) RGs"  
}  
[/code]

Filter the VMs so we are only working with the ones we want, based on $Filter:  
[code language="powershell"]  
write-output "`nGetting filtered VMs from $($RGs.count) RGs using filter $Filter"  
$FilteredVMs = $RGs | Get-AzureRmVM | ? Name -like $Filter  
If(!$FilteredVMs){write-error "Could not find any VMs with a name like $Filter";return} else  
{  
 write-output "Retrieved $($FilteredVMs.count) filtered VMs"  
}  
[/code]

If the days of the week in the database are using a custom format or start the week starts on a different day, we can convert with this simple function:  
[code language="powershell"]  
# 1 = "Sunday"  
# 2 = "Monday"  
# 3 = "Tuesday"  
# 4 = "Wednesday"  
# 5 = "Thursday"  
# 6 = "Friday"  
# 7 = "Saturday"  
Function Convert-DBWeekday {  
 param  
 (  
 [Parameter(Mandatory = $true)]  
 [string] $DBweekday  
 )  
 Switch ($DBweekday) {

1 {  
 $OutputDay = "Sunday"  
 }

2 {  
 $OutputDay = "Monday"  
 }

3 {  
 $OutputDay = "Tuesday"  
 }

4 {  
 $OutputDay = "Wednesday"  
 }

5 {  
 $OutputDay = "Thursday"  
 }

6 {  
 $OutputDay = "Friday"  
 }

7 {  
 $OutputDay = "Saturday"  
 }

}

return $OutputDay

}  
[/code]

Finally, we are looping through all the filtered VMs to get a site specific code in the VM name - this is how we match the opening times to the office/site location stored in the SQL database.

Once we have the site and opening times, we subtract 60 mins from the opening time and add 30 mins to the close time. This provides a small maintenance window for the VM to complete any updates or SCCM software installs.

Then apply the VM Tags. Schedule\_$Weekday  
Obviously a separate Automation runbook is required to read the tags and shutdown/startup the VMs at the time specified in the tags.

[code language="powershell"]  
ForEach($FilteredVM in $FilteredVMs){  
 write-output "`nLooping through each filtered VM. Now using: $($FilteredVM.Name)"

write-output "`nDetermine site code of VM $($FilteredVM.Name)"  
 $Site = ($($FilteredVM.Name) -split "-")[2] -replace '[a-zA-Z]',''  
 If(($Site.length -eq 3)-and($Site -match "\d{3}")){  
 write-output "Site code determined as $Site"  
 } else {  
 write-error "Site code determined as $Site which does not match 3 digits, quitting"  
 return  
 }

# Connect to SQL database and get Site info  
 Write-Output "Getting side ID from SQL database for site $Site"  
 $Site = Read-SqlTableData -ServerInstance $SQLServer -DatabaseName $Database -Credential $DB\_RO -TableName site -SchemaName dbo | ? sitename -like "\*$Site"  
 If($Site.ID){  
 Write-Output "Site ID identified as: $($Site.id)"  
 } else {  
 Send-EmailReport  
 write-error "Could not retrieve $Site from the $Database database."  
 }

Write-Output "`nGetting opening hours for site $($Site.sitename)...."  
 $DefaultHours = Read-SqlTableData -ServerInstance $SQLServer -DatabaseName $Database -Credential $DB\_RO -TableName tblopenhours -SchemaName dbo | ? {$\_.site\_id -eq $($Site.id) -and $\_.usertypecode -eq "Default"}

write-output "`nGetting VMTags from $($FilteredVM.Name)"  
 $VMTags = (Get-AzureRmResource -ResourceId ($FilteredVM).Id -ErrorAction Stop).Tags

ForEach($Day in $DefaultHours){

$StartDate = [datetime]$($Day.starttime)  
 $EndDate = [datetime]$($Day.endtime)  
 $StartTime = get-date $StartDate -Format HH:mm  
 $EndTime = get-date $EndDate -Format HH:mm  
 $WeekDay = Convert-DBWeekday -DBweekday $($Day.weekday)  
 write-output "`nOpening hours for $WeekDay are $StartTime - $EndTime"

write-output "-60 mins from $StartTime and +30 mins to $EndTime"  
 $StartTime = get-date (Get-Date $StartTime).AddMinutes(-60) -Format HH:mm  
 $EndTime = get-date (Get-Date $EndTime).AddMinutes(30) -Format HH:mm

$SchedWeekday = "Schedule\_$Weekday"  
 write-output "Setting tag $SchedWeekday to value: $StartTime-\>$EndTime"  
 $VMTags.$SchedWeekday = "$StartTime-\>$EndTime"

}

write-output "`nWriting VM Tags back to Azure..."  
 Set-AzureRmResource -ResourceGroupName $($FilteredVM.ResourceGroupName) -Name $($FilteredVM.Name) -ResourceType "Microsoft.Compute/VirtualMachines" -Tag $VMTags -Force -Confirm:$false

}

# Send report of job output by email  
Send-EmailReport

[/code]

