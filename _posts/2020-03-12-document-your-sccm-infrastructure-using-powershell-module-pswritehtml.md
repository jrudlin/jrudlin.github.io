---
layout: post
title: Document your ConfigMgr infrastructure using PowerShell module PSWriteHTML
subtitle: Using the PSWriteHTML PoSh module and the ConfigMgr Reporting Services role, build an html diagram of your ConfigMgr server infrastructure.
share-img: "assets/images/sccm-diagram-PSWriteHTML.PNG"
date: 2020-03-12 20:48:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Automation
- PowerShell
- SCCM
- ConfigMgr
tags:
- MEMCM
- SCCM
- ConfigMgr
- Diagram
- Intune
- PowerShell
- PSWriteHTML
- HTML
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![sccm-diagram-PSWriteHTML1]({{ site.baseurl }}/assets/images/sccm-diagram-PSWriteHTML.PNG)

Using the PSWriteHTML PoSh module and the ConfigMgr Reporting Services role, build an html diagram of your ConfigMgr server infrastructure.

* TOC
{:toc}

# Overview

Using some PowerShell modules and connecting to the ConfigMgr Reporting Services point, we will create an interactive HTML diagram of the hierarchy.

This is actually a variation of my original post: [Automatically document your SCCM infrastructure with a Visio diagram using PowerShell](https://jrudlin.github.io/2018/09/08/automatically-document-your-sccm-infrastructure-with-a-visio-diagram-using-powershell/) but instead of using Visio to produce the output diagram of the ConfigMgr infrastructure, we'll use PowerShell and HTML, using the awesome module [PSWriteHTML](https://www.powershellgallery.com/packages/PSWriteHTML) by [EVOTEC](https://twitter.com/evotecpl).

# What does it look like

Check the link to [EVOTEC's](https://twitter.com/evotecpl) blog: [easy-way-to-create-diagrams-using-powershell-and-pswritehtml](https://evotec.xyz/easy-way-to-create-diagrams-using-powershell-and-pswritehtml/) for some ideas on what the output will look like and further details on using the PSWriteHTML module.

This slightly jerky gif shows a small development ConfigMgr hierarchy:

![sccm-diagram-PSWriteHTML2](https://github.com/jrudlin/SCCM-Scripts/blob/master/Document%20Infra/PSWriteHTML.GIF)

# Pre-reqs

 * PowerShell 5.1
 * Internet connectivity from the PC running the script on so it can connect to the PoSh Gallery and install the modules
 * Local admin, as I didn't install the modules into the users profile
 * Network access to the HTTP/HTTPS port of the top level ConfigMgr **Reporting services point** site
 * SCCM Reporting Reader rights with access to these two reports:
   * >Site system roles and site system servers for a specific site
   * >Site status for the hierarchy`
 * A copy of PowerShell script which creates the images needed by the HTML page: [Export images from Visio stencil.ps1](https://github.com/jrudlin/SCCM-Scripts/blob/master/Document%20Infra/Export%20images%20from%20Visio%20stencil.ps1). This is optional, you may want to use your own images to represent the ConfigMgr servers/roles.
 * A copy of PowerShell script which creates the HTML output diagram: [SCCM Document Infrastructure with Interactive html output.ps1](https://github.com/jrudlin/SCCM-Scripts/blob/master/Document%20Infra/SCCM%20Document%20Infrastructure%20with%20Interactive%20html%20output.ps1)

 ## Images

 The output html needs to reference the images of the ConfigMgr servers/roles, if you want things to look professional like :smile:

 These images should stored on a web server. External is best, but if you're only planning on releasing your ConfigMgr diagram internally, then an IIS web server on the LAN is also fine.

The way I retrieved the images for the ConfigMgr roles was to extract them from the Visio stencils. There maybe a better way or an existing repository of images to reference.

 1. Download the [ConfigMgr Visio stencils](https://gallery.technet.microsoft.com/System-Center-Configuration-d67b8ac5)
 1. Edit (Change the `$SCCM_Servers_Stencil_Path` and `$ExportLocation` variables) and then run (As administrator - yeah lazy I know) the [Export images from Visio stencil.ps1](https://github.com/jrudlin/SCCM-Scripts/blob/master/Document%20Infra/Export%20images%20from%20Visio%20stencil.ps1)

You should now have a bunch of images that you can dump onto your http accessible location.

**Note: If you use your own custom images, either update them with same names as the below, or update the script to reflect to the new image names.**

>__Application_Catalog.png
Application_Catalog_WebService_Point.png
Application_Catalog_Web_Site_Point.png
Asset_Intelligence_Synchronisation_Point.png
CAS.png
CAS_Site_Servers_.png
Certificate_Registration_Point.png
Certification_Authority.png
Cloud_Distribution_Point.png
Cloud_management_gateway_connection_point.png
Data_Warehouse_Service_Point_-_CAS.png
Data_Warehouse_Service_Point_-_Primary_Site.png
Distribution_Point_Server.png
Distribution_Point_with_PXE.png
Distribution_Point_with_PXE__State_Migration_Point.png
Domain_Controller.png
EndPoint_Protection_point.png
Enrollment_Point.png
Enrollment_Point__Enrollment_Proxy_Point.81.png
Enrollment_proxy_Point.png
Exchange_20132016_Client_Access_Server.png
Fallback_Status_Point.png
Management_Point.png
Management_Point_with_Device_Management_Point.png
NDES_Server.png
NLB_Software_Update_Point.58.png
Primary_Site_Server.png
Primary_Site_Servers_.png
Protected_Distribution_Point_Server.png
Reporting_Services_Point_.png
SMS_Provider.png
SQL_Cluster_CAS_.png
SQL_Cluster_Primary_Site.png
SQL_Server_-_CAS.png
SQL_Server_-_Database_Replica_for_MP.png
SQL_Server_-_Primary_Site.png
Secondary_Site_Server.png
Secondary_Site__DPPXE.png
Secondary_Site__SUP.png
Service_Connection_Point.png
Software_Update_Point.png
State_Migration_Point.png
System_Health_Validator_Point_.png

# ConfigMgr Infra Diagram

Run [SCCM Document Infrastructure with Interactive html output.ps1](https://github.com/jrudlin/SCCM-Scripts/blob/master/Document%20Infra/SCCM%20Document%20Infrastructure%20with%20Interactive%20html%20output.ps1) with the following params:

 * `-SCCM_SSRS_FQHostname "<Your ConfigMgr top-level SSRS reporting point server fqdn>"`
 * `-ImagesBaseURL "<URI to the http location where the ConfigMgr images stored>"`
 * `-HeaderText "<ConfigMgr Prod Infra>"`

 If the script runs successfully, it should open the HTML ouput at the end:

![sccm-diagram-PSWriteHTML1]({{ site.baseurl }}/assets/images/sccm-diagram-PSWriteHTML.PNG)

