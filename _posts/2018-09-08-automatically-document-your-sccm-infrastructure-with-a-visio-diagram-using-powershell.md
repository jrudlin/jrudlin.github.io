---
layout: post
title: Automatically document your SCCM infrastructure with a Visio diagram using
  PowerShell
date: 2018-09-08 20:50:34.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Automation
- PowerShell
- SCCM
- Visio
tags:
- ConfigMgr
- VBA
- Visio diagram
meta:
  _publicize_done_18837845: '1'
  _wpas_done_18662064: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1038530064453906433";}}
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '21938033407'
  timeline_notification: '1536439835'
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  publicize_linkedin_url: www.linkedin.com/updates?topic=6444295759553798144
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/09/08/automatically-document-your-sccm-infrastructure-with-a-visio-diagram-using-powershell/"
---

[<span style="color:red">Update 2020/03/12:</span> There is now another method that does not require Visio. See post: [Document your ConfigMgr infrastructure using PowerShell module PSWriteHTML](https://jrudlin.github.io/2020-03-12-document-your-sccm-infrastructure-using-powershell-module-pswritehtml/)]

Wow that's a long title. To summarise:  
**PowerShell script = SCCM Visio diagram**

If you have many different SCCM hierarchies in different AD domains and forests to manage, keeping relevant/up to date infrastructure diagrams can be time consuming, especially if there are lots of SCCM site systems, Visio can become painful to use.

I have put this script together: [SCCM Visio diagram.ps1](https://github.com/jrudlin/SCCM-Visio-PoSh/blob/master/SCCM%20Document%20Infrastructure%20with%20Visio%20diagram%20output.ps1) with the help of colleague **Ben Jones** (especially for the maths parts) that **<u>automatically creates a Microsoft Visio diagram of your SCCM / ConfigMgr hierarchy</u>.**

I know there are tools like the CMMap / SMSMap ones, but the SMSMap one in particular is using older shapes, has not been tested with SCCM 2012+ or Visio 2013+. I noticed it was also creating duplicate servers for each role! which is a fairly big overhead when you have 100's of roles.

I wanted to see an open source script that could be used on a schedule to automate the diagram creation. Some further **design requirements** :

- User running script needs **read only** access to the <u>SCCM reporting point.</u> No SQL or SMS Provider/WMI access is required!
- Only HTTP/HTTPS ports are used by the script
- Shapes can easily be swapped out for a new stencil set if required
- Diagram scales from huge to tiny SCCM environments
- Custom shape data can be added to each object from an additional external source (like a corporate CMDB API)

**Pre-reqs before running the script** :

- Microsoft Office Visio 2016 Standard or Professional must be installed on the machine from where the script is run. (Sorry, I know a license is required - I haven't tested with the trial version)
- Download and extract [SCCM Visio stencils](https://gallery.technet.microsoft.com/System-Center-Configuration-d67b8ac5) and then adjust line 12:  

```powershell
# Custom SCCM Stencils used for building the infrastructure diagram  
$SCCM_Servers_Stencil_Path="\\domain.local\shares\files\visio\it\ConfigMgr\1610\Visio\Stencils\v1.3\ConfigMgr 1610 (Servers).vss"  
```

- Network access to the HTTP/HTTPS port of the top level SCCM Reporting services point site
- SCCM Reporting Reader rights with access to these two reports:

```powershell
$SCCM_SiteRoles_ReportName = 'Site system roles and site system servers for a specific site'  
$SCCM_SiteStatus_ReportName = 'Site status for the hierarchy'  
```

- **<u>Adjust the following variable on line 9</u>** to the FQDN of your top level SCCM reporting server:

```powershell 
$SCCM_SSRS_FQHostname = "SCCM-RP.domain.local"; # Central Administration Site reporting point  
```
  
**Some quick notes on the workings of the script** :

- It's using the Visio.Application COM object. You can use the developer mode in Visio to find out more about VBA
- It has only been tested with en-gb language code 2057 in Visio. Try amending line 657 ($visCustPropsLangID) for diff languages.
- The US version of SCCM Reporting Point was used. The names of the two reports mentioned above could be adjusted
- <u>Lines 726-745</u> are for when you want to use a custom API to retrieve additional info about your servers and add it the Visio shape data. I got lazy and didn't add an optional variable to run this or not so you'll probably want to remove this section to avoid errors at the end of the script.

Zoomed out to 30% in Visio - the whole overview. 5 Primary sites and a CAS. ![SCCM Visio Diagram 1]({{ site.baseurl }}/assets/images/sccm-visio-diagram-1.jpg)

Zoomed to about 70% so you can see each site system has the roles, IP and server name as the text box description below the shape: ![SCCM Visio Diagram 2]({{ site.baseurl }}/assets/images/sccm-visio-diagram-2.jpg)

A short YouTube video of how things look as Visio is processing the shapes passed from the PowerShell script: [Automatic Visio diagram video](https://youtu.be/5mBtPIQpDt0)

Thanks to Jean-Sébastien DUCHÊNE for the cool [SCCM Visio stencils](https://gallery.technet.microsoft.com/System-Center-Configuration-d67b8ac5).

And again, huge thanks to **Ben Jones** for his mathematical skills and converting these to .net/PowerShell to get the correct spacing for each shape.

Any questions please ping me on Twitter [@JackRudlin](https://twitter.com/jackrudlin)

