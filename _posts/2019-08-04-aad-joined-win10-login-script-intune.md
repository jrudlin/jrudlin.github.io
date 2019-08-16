---
layout: post
title: Login script for Azure AD Joined, Intune enrolled, Win10 devices
subtitle: PowerShell based login script deployed through Intune. Maps drives for AAD joined Win10 devices - supports AD groups.
share-img: "assets/images/intune-win10-login-script1.png"
date: 2019-08-04 20:48:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Windows 10
- Intune
- PowerShell
- Azure Active Directory
tags:
- Windows 10
- Script
- PowerShell
- Win10
- Azure Active Directory
- AD Groups
- Login script
- AAD Joined
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![CreatedOnDate]({{ site.baseurl }}/assets/images/intune-win10-login-script1.png)

PowerShell based login script deployed through Intune. Maps drives for AAD joined Win10 devices - supports on-prem Active Directory group membership.

## Overview

During a modern desktop design and implementation I decided to push the client down the full Azure AD Joined Windows 10 and Intune route. There was **no SCCM or ConfigMgr** present and only Citrix based group policies. Porting the GPO's to Intune was fairly simple, however the main challenge was maintaing the _legacy drive mappings_ to on-prem file servers.

Again, these Win10 1809 / 1903 devices are AAD Joined. There is no AD Group Policy available. I've seen some other solutions where the AAD Join login script connects to a web api (like an Azure Function) to get the AD group membership of the AAD user, but this seems like a big overhead to me.

I decided the best approach was to maintain a **cloud based login script** which would map the drives based on the **existing AD user groups** using **direct calls to AD** based on the .net class [System.DirectoryServices.AccountManagement.GroupPrincipal](https://docs.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.groupprincipal?view=netframework-4.8) and the GetGroups method.

The components that make up this solution are:

- Configuration/Invoker script which is deployed through Intune and sets an HKLM Registry Run value
- Registry Run value in HKLM which calls the login script on each login for all users
- The login script which is hosted centrally on public Azure Blob storage

## Scripts

Grab a copy of the [login script](https://github.com/jrudlin/Intune/blob/master/Win10-Login-Script.ps1) and the [invoke-login script](https://github.com/jrudlin/Intune/blob/master/Invoke-Win10-Login-Script.ps1).

### Login Script

In the [login script](https://github.com/jrudlin/Intune/blob/master/Win10-Login-Script.ps1) change the global variables in the **Declarations** section at the top to your liking probably the only ones you'll need to change are:

```powershell
$OneDriveFolder = "OneDrive - Org Name"
$CustomHKCU = "HKCU:\Software\Org Name"
```

The rest of the script you'll have to review and update to match your environment. Like the drive mappings section for example:

```powershell
$driveMappingConfig+=  [PSCUSTOMOBJECT]@{
    DriveLetter = "O"
    UNCPath= "\\SUNSRV.$dnsDomainName\saf"
    Description="saf (\\SUNSRV)"
    Group="SUN SRV USERS"
}
```

Some mappings may use a `Group` property which is the corresponding Active Directory security group (containing users).

The way the AD group membership enumeration is working is recursively. I've only had a chance to test it in a small 1000 user environment.
It larger environments it may cause undesired load on the Domain Controllers so please test it carefully in your environment.

### Logging

Logging will be into the users' %temp% folder: **Win10-Login-Script.log**

### Invoke-Login Script

## Azure blob storage

Setup, or use an existing storage account to host the login script in blob storage. V2, LRS, Hot:
![CreatedOnDate]({{ site.baseurl }}/assets/images/intune-win10-login-script2.png)

Create a new container in blob storage with public access level:
![CreatedOnDate]({{ site.baseurl }}/assets/images/intune-win10-login-script3.png)

Upload the [login script](https://github.com/jrudlin/Intune/blob/master/Win10-Login-Script.ps1) and copy its URL
![CreatedOnDate]({{ site.baseurl }}/assets/images/intune-win10-login-script4.png)

## Invoke-Login Script URL

Amend the `$Azure_Blog_Storage_Script_Url` variable in the [invoke-login script](https://github.com/jrudlin/Intune/blob/master/Invoke-Win10-Login-Script.ps1)

## Intune

Deploy your amended invoke-login script using Intune. I went with a simple PowerShell Script item, but you could use a Win32 app with a detection method to increase compliance.
**Don't** deploy using the logged on credentials.
![CreatedOnDate]({{ site.baseurl }}/assets/images/intune-win10-login-script5.png)
