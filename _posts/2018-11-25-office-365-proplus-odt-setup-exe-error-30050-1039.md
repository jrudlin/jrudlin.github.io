---
layout: post
title: Office 365 ProPlus ODT setup.exe error 30050-1039
date: 2018-11-25 17:55:55.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Office 365 ProPlus
tags:
- c2r
- click-to-run
- ODT
- Office 2016
- Office deployment tool
meta:
  _thumbnail_id: '303'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1066752363942285313";}}
  _publicize_job_id: '24638151549'
  timeline_notification: '1543168556'
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  _publicize_failed_18837845: O:13:"Keyring_Error":2:{s:6:"errors";a:1:{s:21:"keyring-request-error";a:1:{i:0;a:6:{s:7:"headers";O:42:"Requests_Utility_CaseInsensitiveDictionary":1:{s:7:"
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/11/25/office-365-proplus-odt-setup-exe-error-30050-1039/"
---
I couldn't find this error blogged anywhere else at the time, so thought I'd do a quick post on it.

When using the click-to-run C2R installer setup.ex from the [Office Deployment Tool](https://www.microsoft.com/en-us/download/details.aspx?id=49117) ODT, I was getting error 30050-1039:

![office proplus error]({{ site.baseurl }}/assets/images/office-proplus-error.png)

This error seemed to appear because of the [RemoveMSI](https://docs.microsoft.com/en-us/deployoffice/configuration-options-for-the-office-2016-deployment-tool#removemsi-element) property in the ODT Configuration.xml:

```xml
<Configuration>  
 <Add OfficeClientEdition="32" Channel="Broad" OfficeMgmtCOM="TRUE">  
  <Product ID="O365ProPlusRetail">  
   <Language ID="en-us" />  
   <ExcludeApp ID="Groove" />  
  </Product>  
 </Add>  
 <RemoveMSI All="True" />  
 <Property Name="FORCEAPPSHUTDOWN" Value="True"/>  
 <Property Name="SharedComputerLicensing" Value="0" />  
 <Property Name="PinIconsToTaskbar" Value="FALSE" />  
 <Display Level="None" AcceptEULA="TRUE" />  
 <Logging Level="Standard" Path="C:\Temp"/>  
</Configuration>
```

In the Office 365 ODT logs I was getting errors like:

> MSIScrub::MSIUninstaller  
> MSI product uninstallation failed with exit code\",\"productId\":\"Office16.VISSTD\",\"exitCode\":\"30066\"

So it was clear from this it wasn't able to uninstall my Visio 2016 Standard installation. I wasn't sure if the packaging team had created un-supported MSI based modification to the Visio Std package, but I didn't want to use the [OffScrubC2R.vbs](https://github.com/OfficeDev/Office-IT-Pro-Deployment-Scripts/blob/master/Office-ProPlus-Deployment/Deploy-OfficeClickToRun/OffScrubc2r.vbs) to workaround the error, I wanted to try and detect and repair the issue on multiple machines in advance.

In the c:\windows\temp\SetupExe(201\*).log I was getting this error:

> [20160] Error: The setup.xml file at: C:\Program Files (x86)\Common Files\Microsoft Shared\OFFICE16\Office Setup Controller\Visio.en-us\setup.xml **does not exist** but is referenced in the ARP entry, therefore transition to MMode is unsafe: VISSTD Type: 27::InstalledProductStateCorrupt.

Strange, the **Visio.en-us** folder didn't exist. I copied the "C:\Program Files (x86)\Common Files\Microsoft Shared\OFFICE16\Office Setup Controller\\**VISSTD**" folder and renamed it to **Visio.en-us** and re-ran the ODT setup.exe.

Next error in the SetupExe(201\*).log:

Error: The install state of the ProductCode **{90160000-0054-0409-0000-0000000FF1CE}** , which is referenced in the ProductCodes registry value, is not INSTALLSTATE\_DEFAULT. The install state state is -1. Therefore transition to MMode is unsafe for product: VISSTD Type: 27::InstalledProductStateCorrupt.

Checking the Registry key:
> HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Office16.VISSTD I could see the **{90160000-0054-0409-0000-0000000FF1CE}** ProductCode was listed:

![O365 proplus install 3]({{ site.baseurl }}/assets/images/o365-proplus-install-3.png)

I removed it from the list. Now you may think this would effect the uninstall of this MSI (which in my case was the core Visio install), but it didn't.

I re-ran the ODT setup.exe and the previous Office install (Office 2016 Professional MSI edition) was completely uninstalled without error, finally.
