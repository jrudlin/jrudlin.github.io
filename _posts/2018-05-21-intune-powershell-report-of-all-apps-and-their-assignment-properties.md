---
layout: post
title: Intune - PowerShell report of all apps and their assignment properties
date: 2018-05-21 20:41:33.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Azure
- Intune
- PowerShell
tags:
- App Assignment report
- Microsoft Graph API
meta:
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:56:"https://twitter.com/JackRudlin/status/998665128697622528";}}
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=13673999&stype=M&topic=6404430823294062592&type=U&a=tAzM
  _publicize_done_18837845: '1'
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '18102846387'
  timeline_notification: '1526935294'
  _wpas_done_18662064: '1'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/05/21/intune-powershell-report-of-all-apps-and-their-assignment-properties/"
---
Well, I figured this was available in the Intune Portal already, but maybe I missed it?  
I then kinda assumed I would be able to find a script already loaded in GitHub. I found most of it but not exactly what I was after.

In SCCM, you can easily get a report of all Apps and where they are deployed or you can look at your collections and see which deployments are active. This isn't so easy in Intune. Time for the Microsoft Graph API and some PowerShell:

90% of this work was already and is available here: [Microsoft Graph examples on Github](https://github.com/microsoftgraph/powershell-intune-samples/tree/master/Applications)

I just needed some fluff at the end to put it into a nice table and add a few missing properties along the way.

Pre-reqs:

1. AzureADPreview module version 2.0.1.11
2. Intune read rights and read access to Azure AD groups
3. Microsoft Intune Powershell app in AAA (script function creates this)

Take the following 3 functions and the region authentication section from this script: [Application\_Get\_Assign.ps1](https://github.com/microsoftgraph/powershell-intune-samples/blob/master/Applications/Application_Get_Assign.ps1) and paste them into your new PowerShell script.

```powershell
# Retrieve all apps from the tenant
$apps = Get-IntuneApplication
Write-Host "Retrieved $($apps.count) apps" -ForegroundColor Green
 
# Create a new array object
$Output=New-Object System.Collections.ArrayList
 
ForEach($App in $Apps){
    Write-Host "`nGetting assignments for app: $($app.displayname)" -ForegroundColor Yellow
    $AppID = $app.id
 
    $graphApiVersion = "Beta"
    $Resource = "deviceAppManagement/mobileApps/$AppID/?`$expand=categories,assignments"
    $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
 
    $AppQuery = (invoke-RestMethod -Uri $uri –Headers $authToken –Method Get)
 
    If(($AppQuery.assignments -eq $null) -or ($AppQuery.assignments -eq "") -or ($AppQuery.assignments.count -lt 1)){
 
            Write-Host "No assignments for this app" -ForegroundColor Yellow
 
    } else {
 
        #Write-Host "Platform odata: $($AppQuery.'@odata.type')"
 
                # The many diff types of app in Intune, we switch the variable to the correct platform
 
        $Platform = switch -Wildcard ( $AppQuery.'@odata.type' )
        {
            *androidForWorkApp* { 'Android for Work' }
            *microsoftStoreForBusiness* { 'Microsoft Store' }
            *iosVppApp* { 'Apple VPP' }
            *windowsPhoneXAP* { 'Windows Phone XAP'}
            *webApp* { 'Web Link'}
            default { 'Unknown' }
        }
 
        ForEach($assignment in $AppQuery.assignments){
            # Available or Required
            write-host "Assignment intent: $($assignment.intent)"
 
            If ($($assignment.target.'@odata.type') -like "*allLicensedUsersAssignmentTarget"){
                Write-Host "Published to All Users"
                $GroupName = "All Users"
            } else {
                # Lookup the AAD Group displayname
                write-host "Group ID: $($assignment.target.GroupID)"
                $GroupName = (Get-AzureADgroup -ObjectId $assignment.target.GroupID).DisplayName
            }
 
            # Add all the properties into a new object in the array
            Write-Host "Group Name: $GroupName"
            $Output.Add( (New-Object -TypeName PSObject -Property @{"Name"="$($app.displayname)";"Group" = "$GroupName";"Assignment" = "$($assignment.intent)";"Platform" = "$Platform"} ) )
 
        }
 
    }
 
}
 
# Format the column order by modifying the table output
$output | select Name,Group,Assignment,Platform
```

When you run the script, you will get console debug output during the script runtime:

![Intune1]({{ site.baseurl }}/assets/images/intune1.jpg)

Then at the end you get the full table:

![Intune21]({{ site.baseurl }}/assets/images/intune21.jpg)

So for a quick overview of all your Intune app deployments (oops, I mean assignments), this script provides a simple table that you can use in documentation or to keep tabs on whats deployed where.

Remember, the table only lists the apps that have an assignment. Apps without assignments can be seen in the debug output in the console window.

