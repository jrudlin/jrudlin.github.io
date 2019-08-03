---
layout: post
title: Renaming Teamviewer portal devices with the primary user from Intune
subtitle: PowerShell script with functions that append the devices' [primary user] (retrieved from Intune) to the device name in the Teamviewer portal
share-img: "assets/images/rename-teamviewer-device1.png"
date: 2019-07-30 20:48:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- TeamViewer
- Intune
- PowerShell
- Graph
tags:
- Intune
- Script
- PowerShell
- TeamViewer
- Intune Primary User
- Intune Graph
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device1.png)

PowerShell script with functions that append the devices' [primary user] (retrieved from Intune) to the device name in the Teamviewer portal.

## TeamViewer Portal

By default, when a device is registered in the TeamViewer portal by deploying the msi (through Intune of course) it doesn't contain any information regarding the primary (or any other) _user of the device_. Generally, the end user will not know the name of their device and the IT Service Desk would prefer to search by the users' name rather than device.

By scheduling an Azure Automation script that uses the TeamViewer and Intune Graph API, we can append a `[User Name]` to the device so it looks like: `Lap-Win10-1 [Jack Rudlin]`

See below on how to set things up.

## TeamViewer token

To connect to the TeamViewer API and add the primary user to the existing device name, we need a token to authenticate.

Login to the [TeamViewer portal](https://login.teamviewer.com/LogOn) with an admin account that will remain active - maybe a generic super user? In my example I'm using an admin account with my name.

> You have to create a token under a user account, rather than the company profile, because only user accounts have access to edit devices via the API.

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device2.png)

_Edit_ your account profile in the top right drop down menu:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device3.png)

Create a new script token from the **Apps** menu:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device4.png)

Give the token a name, description and **edit** access to **Computers**:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device5.png)

Make a note of the variable - however it can always be retrieved later by editing the token:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device6.png)

## The script

### Github Source
[Rename-TeamViewerDevice.ps1](https://github.com/jrudlin/Intune/blob/master/Rename-TeamViewerDevice.ps1)

### PoSh Modules
The script relies on PowerShell module [Microsoft.Graph.Intune](https://www.powershellgallery.com/packages/Microsoft.Graph.Intune) so make sure that's installed in your Azure Automation account and is running at least module version 6.1907.1.

**Note:** There is a bug with older versions of the Microsoft.Graph.Intune module in Azure Automation Accounts. More details [here](https://github.com/Microsoft/Intune-PowerShell-SDK/issues/25).

### Authentication
Use an account or AAD Application that has read-only access to Intune device properties.
See below for variables to amend.

### Variables

Replace `$token` variable with the TeamViewer token that you created above.

```powershell
#Define Auth Token for TeamViewer
$token = "3462611-habtKKAf1VrtYJD2UWZO"
```

Replace `$IntuneROAccount` with the name of the Azure Automation Account credentials that will provide RO access to Intune.

You don't have to, but if you want to change the format of how the user names will appear and how its matched by the script, change the `$NamingPatternMatch` variable and also the `[$PrimaryUser]` on line 141.

```powershell
$NamingPatternMatch = '\[(.+)\]'

[$PrimaryUser]
```

### Output

During the script run, output will look like this if running from Azure Automation:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device8.png)

Or this from a shell:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device7.png)

`true` is returned when a TeamViewer device is succesfully renamed.

The result looks like this from the TeamViewer portal and allows **simple search by users name** from the bar in the top right:

![CreatedOnDate]({{ site.baseurl }}/assets/images/rename-teamviewer-device1.png)