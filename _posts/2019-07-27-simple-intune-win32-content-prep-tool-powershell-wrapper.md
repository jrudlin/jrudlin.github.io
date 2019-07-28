---
layout: post
title: Simple PowerShell wrapper for the Intune Win32 Content Prep tool exe
subtitle: Super simple PowerShell script/wrapper for the IntuneWinAppUtil.exe content prep tool to speed up your Intune packaging pipeline
share-img: "assets/images/Package-IntuneApp1.png"
date: 2019-07-27 20:47:55.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Intune
- PowerShell
tags:
- Packaging
- Intune
- Script
- PowerShell
- intunewin
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![CreatedOnDate]({{ site.baseurl }}/assets/images/Package-IntuneApp1.png)

Super simple PowerShell script/wrapper for the IntuneWinAppUtil.exe content prep tool to speed up your Intune packaging pipeline.

## Intune Win32 wrapper script

Just a quick post this time - more of a reminder for myself if anything :smiley: so I don't have to remember the parameters to pass to IntuneWinAppUtil.exe, instead I just _CTRL+Space_ or _Tab_ in PowerShell.

If you use the IntuneWinAppUtil.exe [content prep tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool) to create Win32 apps for deployment through Intune, I created this simple wrapper to speed things up.

Drop the [Package-IntuneApp.ps1](https://github.com/jrudlin/Intune/blob/master/Package-IntuneApp.ps1) into the root of your packaging directory structure, mine looks like this:

- OneDrive\Packages
  - Packages\Microsoft-AzureStorageExplorer
  - Packages\Chocolatey-Free
  - Packages\DNSFilter-Win10-Agent
  - Packages\Python
  - Packages\Package-IntuneApp.ps1

```powershell
.\Package-IntuneApp.ps1 -SourceFolder .\Microsoft-AzureStorageExplorer\ -SetupFile Install-AzStorageExplorer-Choco.ps1
```

Specify the _-SourceFolder_ (The package folder containing your installation binaries/scripts) and the _-SetupFile_ which is the file Intune will run on the endpoint to start the installation.

You an also specify the _-OutPutFolder directory if you want to change it from the default **_Output** folder.

![CreatedOnDate]({{ site.baseurl }}/assets/images/Package-IntuneApp2.png)
![CreatedOnDate]({{ site.baseurl }}/assets/images/Package-IntuneApp3.png)

That's all folks!