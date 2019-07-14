---
layout: post
title: Modifying Group Policy Preferences Registry collections with PowerShell
date: 2017-08-12 17:09:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Group Policy
- PowerShell
tags:
- XML
meta:
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '8198681114'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2017/08/12/modifying-group-policy-preferences-registry-collections-with-powershell/"
---
Ok guys, loosely sticking with the VDI infrastructure in Azure topic, I needed some automation around Group Policies - seeing as VDI is all about automation and scalability, I didn't want the solution being hindered by too many manual processes (like maintaining GPO's for new sites/VMs etc).

The VDI solution is for libraries in a London borough and is accessible to members of the public using their library card to authenticate on Insight Media's iCAM platform. The thin client's need to startup and automatically log into an assigned desktop VM in Azure. The VMs are Windows 10 1607 CBB 64-bit.

As we all know, first time login to Windows10/8 is quite slow as it creates the appx packages for the Start Menu, but the main reason for the auto-login is iCAM.

iCAM locks out the Windows desktop until a user enters their library card#, but it first needs to load the explorer shell so that the iCAM app can then be launched. It's the not greatest app, but that's for later.

As the Azure VMs are deployed using Azure Automation, I need to create the appropriate Group Policy so that each VM automatically logs into Windows with a specific AD user account.

I have written a PowerShell function that updates Group Policy Preferences XML with Registry preferences.

The function clones an existing Registry collection and edits the properties with username/password/VM name etc. and saves it back to the domain.

This is the screenshot of the Reg collection I'm cloning and modifying: VM-L270-001

![Reg GPO Prefs collection]({{ site.baseurl }}/assets/images/reg-gpo-prefs-collection.jpg)

[code language="powershell"]  
param  
(  
[Parameter(Mandatory = $false)]  
[string] $GPOName = "Win10 VDI-Computers",\</code\>

[Parameter(Mandatory = $false)]  
[string] $VMName = "VM-270-002",

[Parameter(Mandatory = $false)]  
[string] $UName,

[Parameter(Mandatory = $false)]  
[string] $PWord,

[Parameter(Mandatory = $false)]  
[string] $TemplateVMName = "VM-270-001"  
)

If(-not(get-module -Name GroupPolicy)){  
write-host "Importing GroupPolicy Module"  
Import-Module GroupPolicy  
}

$GPO = Get-GPO -Name $GPOName  
 $GPOGuid = ($GPO.Id).Guid  
 $Domain = $GPO.DomainName  
 $GPOPath = "\\$Domain\SYSVOL\$Domain\Policies\{$GPOGuid}\Machine\Preferences\Registry\Registry.xml"  
 $Site = ($VMName -split "-")[2] -replace '[a-zA-Z]',''  
 $TemplateSite = ($TemplateVMName -split "-")[2] -replace '[a-zA-Z]',''

#Retrieve XML  
 $xml = [xml](Get-Content $GPOPath)

If($xml){  
 write-output "XML retrieved from $Domain"  
 } else {  
 Write-Error "XML could not be retrieved from $Domain"  
 }

# Check if collection for the specified site exists  
 $SiteColUserAutoLoginVM = ((((($xml.ChildNodes).Collection | ? Name -EQ "iCAM Client VDI").Collection | ? Name -like "$Site\*").Collection | ? Name -Like "User Auto Login").Collection | ? Name -Like $VMName)

If($SiteColUserAutoLoginVM){  
 Write-Output "Site collection $Site exists and contains 'User Auto Login' with a sub collection $VMName already"  
 } else  
 {  
 Write-Output "No 'User Auto Login' collection with name $VMName for $Site found. Copying template $TemplateVMName Site collection...."

# Many arrays in the GPO as it has a lot of collections (folders) for GPPref filtering  
 # Get index of the icam client collection  
 $ClientCollections = [Collections.Generic.List[Object]]($xml.DocumentElement.Collection)  
 $indexclient = $ClientCollections.FindIndex( {$args[0].Name -eq "iCAM Client VDI"} )

# Get index of the site collection  
 $SiteCollections = [Collections.Generic.List[Object]]($xml.DocumentElement.Collection[$indexclient].Collection)  
 $indexsite = $SiteCollections.FindIndex( {($args[0].Name -like "$site\*")} )  
 $indexsiteTemplate = $SiteCollections.FindIndex( {($args[0].Name -like "$TemplateSite\*")} )

# Get template VM collection  
 $TemplateSiteVMCols = [Collections.Generic.List[Object]]($xml.DocumentElement.Collection[$indexclient].Collection[$indexsiteTemplate].Collection.Collection)  
 $indexVMTemplate = $TemplateSiteVMCols.FindIndex( {($args[0].Name -like "$TemplateVMName")} )

# Clone the 270 site node  
 $NodeToClone = $xml.RegistrySettings.Collection[$indexclient].Collection[$indexsiteTemplate].Collection.Collection[$indexVMTemplate].Clone()

# Modify the XML before comitting it  
 $NodeToClone.name = $VMName  
 $NodeToClone.uid = [string]"{$((new-guid).Guid)}".ToUpper()  
 $NodeToClone.Filters.FilterComputer.name = $VMName  
 $NodeToClone.Registry[0].uid = [string]"{$((new-guid).Guid)}".ToUpper()  
 $NodeToClone.Registry[0].Properties.value = $PWord  
 $NodeToClone.Registry[1].uid = [string]"{$((new-guid).Guid)}".ToUpper()  
 $NodeToClone.Registry[1].Properties.value = $UName

# Add the clone to the xml  
 If($xml.DocumentElement.Collection[$indexclient].Collection[$indexsite].Collection){  
 write-output "Writing cloned template to new site..."  
 $xml.DocumentElement.Collection[$indexclient].Collection[$indexsite].Collection.AppendChild($NodeToClone)  
 } else {  
 write-error "Please create 'User Auto Login' registry collection under site $site in GPO: $GPOName"  
 }  
 $xml.Save($GPOPath)

}

[/code]

