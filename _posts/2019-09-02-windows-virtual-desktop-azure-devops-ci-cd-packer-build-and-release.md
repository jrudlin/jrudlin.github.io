---
layout: post
title: Windows Virtual Desktop CD/CD image using Azure Devops and Packer
subtitle: Using Azure Devops CI/CD Pipelines, PowerShell and Packer, create Builds and Releases for a scale-out Win10 1903 WVD deployment in Azure.
share-img: "assets/images/wvd-azure-devops1.jpg"
date: 2019-09-02 20:48:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Windows 10
- Azure Devops
- WVD
- Windows Virtual Desktop
- Continuous Integration
- Continuous Delivery
- PowerShell
- Virtual Machine
- Packer
tags:
- Azure VM Image
- Managed Disks
- PowerShell
- Storage Account
- OS Disk
- Packer
- Json
- Devops
- CI/CD
- Win10 Multi-user
- ARM Templates
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![DevopsOverview]({{ site.baseurl }}/assets/images/wvd-azure-devops1.jpg)

Using Azure Devops CI/CD Pipelines, PowerShell and Packer, create Builds and Releases for a scale-out Win10 1903 WVD deployment in Azure.

* TOC
{:toc}

# Overview

Ok, it's difficult for me to explain what this post is about using just the title and subtitle so I'll try and do a better job here.

With [Windows Virtual Desktop](https://docs.microsoft.com/en-gb/azure/virtual-desktop/overview) being close to GA now, I wanted to put a continuous build solution together for the **Windows 10 Azure VM Image** which will be used to build the **WVD solution** from. In English this means we will be building a traditional **Windows 10 gold image** with all our apps and static config etc. and then using this image to deploy Azure VMs from, forming the Windows Virtual Desktop Host Pools for RemoteApps.

The **gold image** will contain all applications, installed using Packers' **PowerShell Provisioner**. Not only is a Build Pipeline used but a Release Pipeline is then run to deploy the WVD solution to Azure using this latest Build image artifacts and the WVD ARM Template.

As a regular consultant for SCCM/ConfigMgr/Intune, using **Devops + Packer + Azure** to build Win10 machines from a gold image seems similar in approach to SCCM Task sequences (B&C, Deployment), however rather than using the Microsoft VLSC ISO as the base to start with, here we have image transform steps like:

> Azure Marketplace Win10 EVD image > Azure VM Managed Image > WVD

A close alternative to the [HasiCorp Packer](https://www.packer.io/) steps in the Azure Devops Pipelines, as described in this post, is the preview service from Microsoft: [Azure Image Builder](https://docs.microsoft.com/en-gb/azure/virtual-machines/windows/image-builder-overview) which is actually based on Packer as well - only in Azure Image Builder we can submit the whole job as an ARM Template - I will hopefully get time to test this preview service out soon and issue another writeup :smile:.

So before you get to the TL;DR section, these are some of the cool things I'm going to use to put this solution together:

* Azure Devops Pipelines (Build & Release)
* HashiCorp Packer
* Azure VM Managed Image (not VHD)
* Azure Key Vault
* Azure Devops Custom Host Agent (running on an Azure VM)
* Azure File Share (for application installation)
* ARM Template for Windows Virtual Desktop Host Pool provisioning

Most of the ideas for this solution came from [@samcogan](https://twitter.com/samcogan)'s post: [Building Packer Images with Azure DevOps](https://samcogan.com/building-packer-images-with-azure-devops/). Thank's Sam.

# Azure Devops

## Pre-reqs / Requirements

* Access with **Basic** Access Level at least to an [Azure Devops](https://dev.azure.com) organisation and/or project. Stakeholder level doesn't allow Pipelines to be used!

**Note:** There is also a preview feature to allow [Stakeholders access to pipelines](https://docs.microsoft.com/en-us/azure/devops/organizations/security/provide-stakeholder-pipeline-access?view=azure-devops) 

* **Owner** access to an Azure Subscription so you can create Resource Groups, VMs, Key Vaults, Images and a Subscription level service principal
* Windows Virtual Desktop **tenant owner** permissions
* Azure AD **Global Admin** access, or access to create new service principals

## Devops Project

### Outline

The Azure Devops Build Pipeline will be used to run Packer, which takes an Azure Marketplace Win10 1903 EVD image (with or without O365 ProPlus) and builds a VM from it. Once the VM is provisioned, Packer PowerShell Provisioner will connect to an Azure File Share and begin to install your business applications. Once the custom config/apps are finished, Packer will sysprep, shutdown and convert the VM to an Azure VM Image - following this, all the other resources are cleanly removed.

### Devops Project

Start off by creating a new Project in Azure Devops:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops2.jpg)

Give the Project a **name**.

Choose **Private**, unless your Org allows **public projects**, in which case there are benefits like **extra runtime minutes for the hosted agents** - which is extremely handly for long running Build tasks.

I expect you'll want to use **Git** based version control, but it doesn't matter for the purposes of this project, same goes for work item process - **basic** is fine.

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops3.jpg)

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops4.jpg)

### Repo

You can Clone the repo into VSCode if you wish, I'm going to show the process using the Devops web portal.

#### Initialise Repo

**Initialise the Repo** that gets created automatically:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops5.jpg)

#### Folders

Create two new folders:
* **Packer Build - Win10 1903 EVD**
* **ARM Templates**

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops6.jpg)

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops7.jpg)

Commit the Readme in the new folder:
![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops8.jpg)

Repeat for the **ARM Templates** folder:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops9.jpg)

#### ARM Template

Using guidance here: [Create a host pool with an Azure Resource Manager template](https://docs.microsoft.com/en-us/azure/virtual-desktop/create-host-pools-arm-template), download the WVD ARM template from [Github](https://github.com/Azure/RDS-Templates/tree/master/wvd-templates).

Even though I already have an existing WVD Host Pool (created using the Azure Portal originally) I am using the [Create and provision new Windows Virtual Desktop hostpool](https://github.com/Azure/RDS-Templates/blob/master/wvd-templates/Create%20and%20provision%20WVD%20host%20pool/mainTemplate.json) template rather than the [Update Existing Windows Virtual Desktop Hostpool](https://github.com/Azure/RDS-Templates/blob/master/wvd-templates/Update%20existing%20WVD%20host%20pool/mainTemplate.json) template. Reason being, it fits better with the release process in my opinion. I don't want to deallocate or delete the existing Host Pool VMs (which is what the latter template does) until I'm satisfied the new VMs are working.

Upload the [template](https://github.com/Azure/RDS-Templates/blob/master/wvd-templates/Create%20and%20provision%20WVD%20host%20pool/mainTemplate.json) to the **ARM Templates** folder:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops10.jpg)
![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops11.jpg)

Now we need to create a default parameters.json template. The most common defaults for the WVD properties will be stored here, things that hardly ever change like **domain name** - but don't worry, all these properties can be overridden individually during the Build process.

Copy the `$schema`, `contentVersion` and `parameters` sections from the **mainTemplate.json** into a new **parameters.json** file.

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops12.jpg)

Don't forget the formatting at the end:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops13.jpg)

Upload the **parameters.json** into the **ARM Templates** folder:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops14.jpg)

**Edit** the **parameters.json** file:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops15.jpg)

I've just changed default Managed Disk type to Standard, in case someone forgets to override the disk type later on, the solution will build with the cheapest disks instead of the most expensive :relieved::

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops16.jpg)

**Commit** your changes.

#### Packer Template

Now lets get an example copy of a Packer template for Azure based on using an existing image: [windows_custom_image.json](https://github.com/hashicorp/packer/blob/master/examples/azure/windows_custom_image.json)

Save the file to disk as **packer-win10_1903.json** and upload it to the **Packer Build - Win10 1903 EVD** folder:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops17.jpg)

The file structure should like the following:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops18.jpg)

The repo and template files structure is now complete.
We will need to edit the Packer template now, but the WVD ARM template should remain static.

### Packer VM Build Service Principal

Packer will need an Azure Service Principal in the Azure subscription where the WVD machines will be built. Packer creates a Resource Group, Key Vault, VM, Storage and networking during each Build - which it then deletes at the end, after the VM Image has been successfully created. My Service Principal has **Contributor** access at the Subscription level but you can of course allocate the individual roles instead.

[How to: Use the portal to create an Azure AD application and service principal that can access resources](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)

Get the **Application ID** and the **Secret** of the new Service Principal. Keep these handy for a while (but not saved to disk) as we'll be using them a few times.

In your Repo, edit the **packer-win10_1903.json** and remove the values from the `client_id` and `client_secret` variables (This is important - don't hardcode this secret info into variables - we will use Azure Key Vault later). It should look like this:

```json
"client_id": "",
"client_secret": "",
```

### Azure File Share

We'll use an **Azure File Share** which will be **mapped** as an **SMB drive** from Windows during the Packer build process. This will host the binaries/packages for all the application installs that will go into the gold image (Azure VM Image).

You can create the [Azure File Share](https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-create-file-share) in an existing Azure Storage Account.

Once your File Share is created, upload all your application install sources:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops33.jpg)

Ii you click on **Connect** you'll be able to copy and paste the UNC path for the share that we'll need to our Devops variable later:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops34.jpg)

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops35.jpg)

Finally, get the **Storage Account name** and **Access Key** so that we can store it in the Key Vault later on.

### WVD Security

#### WVD Service Principal

The WVD Service Principal will have RDS Owner rights so that when we deploy the Azure Image created with Packer with the ARM template, the VMs will join the WVD Host Pool using the Service Principal creds.

If you don't already have one, create a Service Principal as a Windows Virtual Desktop RDS Owner: [Tutorial: Create service principals and role assignments by using PowerShell](https://docs.microsoft.com/en-us/azure/virtual-desktop/create-service-principal-role-powershell)

Use these sections:

* Create a service principal in Azure Active Directory
* Create a role assignment in Windows Virtual Desktop

#### WVD Domain Join Creds

We will need the Active Directory user UPN and password of the account that access to join the Win10 WVD VMs to the Active Directory domain.

### Key Vault

**Create a new** or use an **existing** [Azure Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/quick-create-portal#create-a-vault) for the purposes of **storing our secrets** that will be used during the Build and Release Pipelines.

There are different methods for retrieving and using secure variables in Azure Devops, one way is to query the Key Vault directly, each time a Pipeline task runs, like in this post: [Using secrets from Azure Key Vault in a pipeline](https://www.azuredevopslabs.com/labs/vstsextend/azurekeyvault/)

In my example here, we'll **sync the secrets** from the Key Vault into an Azure Devops Variable Group and retrieve the secrets from that.

#### Secrets

Once you have chosen which **Key Vault** you will store your secrets in, go ahead and add the following **Secrets**:

* Azure File Share Storage Account Name / Storage Account Access Key
* Azure Subscription Service Principal Application ID / Secret
* Active Directory Domain Join UPN / Password
* WVD Tenant Admin (Service Principal) Application ID / Secret

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops27.jpg)

### Devops Variable Group - Key Vault

In your WVD project > Pipelines > Library, create a new Variable Group:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops19.png)

* Enable the Azure key vault option.
* Select your subscription from the drop-down menu.
  * At first I didn't see my subscriptions. I need to grant the account I was using in Devops read access to the Azure subscriptions.
* Don't authorize - as this will try and create a new Service Principal..
* Click the drop-down next to Authorize and choose **Advanced options**.

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops20.png)

Click on the link for **Use the full version of the service connection dialog**:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops21.png)

Using Service Principal details (the one you created just now) fill in the **Service principal client ID** - this is the **Application ID**. Fill in the **Service principal key** - this is the **Secret**
These will map to the Packer variables `client_id` and `client_secret` later on.

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops22.jpg)

**Verify** the connection to confirm the Service Principal has access to the subscription, then click Ok:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops23.jpg)

Select the Key Vault that you will use to store the variables/secrets:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops24.jpg)

**Add** the variables from the Key Vault in Devops:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops25.jpg)
![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops26.jpg)

**Save** the new Variable Group:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops28.jpg)

The secrets are now ready to be securely access from the Build and Release pipelines:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops29.jpg)

### Devops Variable Group - Devops

Not all the variables used need to be stored in an Azure Key Vault. It's simpler for variables that don't require encryption, to be stored in standard Devops Variables Groups.

In your WVD project > Pipelines > Library, create a new Variable Group:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops30.jpg)

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops31.jpg)

Add the following new variables and their associated values.

* Az Subscription ID and AAD Tenant ID are obvious.
* Packaged App path is the UNC to the Azure File Share where the app installs are located.
* The final variable is the name of the Resource Group where the gold image for WVD will be stored after the Packer build completes. **This Resource Group must already exist**.

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops32.jpg)

### Packer Template - Detailed

#### Object ID

Seems like the Packer template on Github has an undocumented and unrequired variable/builder property. Go ahead and **delete** `object_id` references from the **Variables** and **Builders** sections:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops36.png)

#### More Secrets - Azure Files

Earlier we edited the Packer template to support the **client_id** and **client_secret** secure variables for our Service Principal.

For the Packer build, the only other secure variables we'll use are the Azure Storage Account name (the one that contains the Azure File Share as detailed above) and the Access Key for said Storage Account.

**Edit** the Packer template: **packer-win10_1903.json** and **add** two new empty `variables`:

```json
"AppInstallsStorageAccountName": "",
"AppInstallsStorageAccountKey1": "",
```

So the variables section should now look like this:

```json
  "variables": {
    "client_id": "",
    "client_secret": "",
    "AppInstallsStorageAccountName": "",
    "AppInstallsStorageAccountKey1": "",
    "resource_group": "{{env `ARM_RESOURCE_GROUP`}}",
    "storage_account": "{{env `ARM_STORAGE_ACCOUNT`}}",
    "subscription_id": "{{env `ARM_SUBSCRIPTION_ID`}}"
  },
```

#### Packer Custom Variables

Packer can accept variables passed from the Azure Devops tasks as environment variables. Don't hard code any values, we will be passing the values (mostly from the Variable Group) later on when we the Build and Release pipelines.

Checking the [Packer build configuration reference](https://www.packer.io/docs/builders/azure.html) we can see that if we're generating an Azure VM Managed Image like this:

![WVDAzureDevops]({{ site.baseurl }}/assets/images/wvd-azure-devops37.jpg)

then we need to use some different properties in the json template.

In the `variables` section, delete lines:

```json
"resource_group": "{{env `ARM_RESOURCE_GROUP`}}",
"storage_account": "{{env `ARM_STORAGE_ACCOUNT`}}",
```

and replace with:

```json
"wvd_goldimage_rg": "{{env `wvd_goldimage_rg`}}",
"az_tenant_id": "{{env `az_tenant_id`}}",
"packaged_app_installs_path": "{{env `packaged_app_installs_path`}}",
"Build_DefinitionName": "{{env `Build_DefinitionName`}}",
"Build_BuildNumber": "{{env `Build_BuildNumber`}}"
```

The variables section now looks like this:

```json
  "variables": {
    "client_id": "",
    "client_secret": "",
    "AppInstallsStorageAccountName": "",
    "AppInstallsStorageAccountKey1": "",
    
    "wvd_goldimage_rg": "{{env `wvd_goldimage_rg`}}",
    "az_tenant_id": "{{env `az_tenant_id`}}",
    "subscription_id": "{{env `ARM_SUBSCRIPTION_ID`}}",

    "packaged_app_installs_path": "{{env `packaged_app_installs_path`}}",

    "Build_DefinitionName": "{{env `Build_DefinitionName`}}",
    "Build_BuildNumber": "{{env `Build_BuildNumber`}}"
  },
```

**Explanation:**
* `"resource_group"` and `"storage_account"` variables are only required for VHD type builds so we don't need those as we're building a VM Managed Image.
* `"wvd_goldimage_rg"` is the Resource Group where the gold image that Packer creates will be stored - it must exist already.
* `"az_tenant_id"` the AAD Tenant linked to the Azure subscription the resources will be deployed in.
* `"packaged_app_installs_path"` This will be the UNC path to the Azure File Share created earlier.
* `"Build_DefinitionName"` and `"Build_BuildNumber"` are builtin Devops variables taken from the properties of the Build pipeline. These are used to name the Azure VM Managed Image.


#### Packer Builder Properties

You may have noticed that we just deleted the `"resource_group"` and `"storage_account"` variables from the `variables` section, so naturally we need to do the same in the `builders` section too, as well as removing some others:

**Edit** the Packer template again (**packer-win10_1903.json**) and **delete** the following lines from the `builders` section: 

```json
"capture_container_name": "images",
"capture_name_prefix": "packer",
"image_url": "https://my-storage-account.blob.core.windows.net/path/to/your/custom/image.vhd",
```

My `builders` now looks like this:

```json
"builders": [
  {
    "type": "azure-arm",
      
    "client_id": "{{user `client_id`}}",
    "client_secret": "{{user `client_secret`}}",
    "subscription_id": "{{user `subscription_id`}}",

    "os_type": "Windows",
    
    "azure_tags": {
      "dept": "engineering",
      "task": "image deployment"
    },

    "location": "West US",
    "vm_size": "Standard_DS2_v2"
  }
],
```

Now for adding a whole bunch of new properties to the Packer `builders`. I won't go through each one individually here, as you can lookup the details in the [Packer build configuration reference](https://www.packer.io/docs/builders/azure.html).

Add the following `builders`:

```json
"tenant_id": "{{user `az_tenant_id`}}",

"managed_image_name": "{{user `Build_DefinitionName` | clean_image_name}}-{{isotime \"2006-01-02-1504\"}}-Build{{user `Build_BuildNumber`}}",
"managed_image_resource_group_name": "{{user `wvd_goldimage_rg`}}",

"image_publisher": "MicrosoftWindowsDesktop",
"image_offer": "Windows-10",
"image_sku": "19h1-evd",
"communicator": "winrm",
"winrm_use_ssl": "true",
"winrm_insecure": "true",
"winrm_timeout": "3m",
"winrm_username": "packer",

"managed_image_storage_account_type": "Premium_LRS",
"temp_resource_group_name": "rg-PackerBuild-Prod-1",
"virtual_network_name": "VNET-PROD-1",
"virtual_network_subnet_name": "Subnet-PackerImage-Prod-1",
"private_virtual_network_with_public_ip": "True",
"virtual_network_resource_group_name": "rg-VNET-Prod-1",
"azure_tags": {
    "Project": "Packer IT Image"
},
"async_resourcegroup_delete":true
```

My `builders` now looks like this:

```json
"builders": [
  {
    "type": "azure-arm",
      
    "client_id": "{{user `client_id`}}",
    "client_secret": "{{user `client_secret`}}",
    "tenant_id": "{{user `az_tenant_id`}}",
    "subscription_id": "{{user `subscription_id`}}",

    "os_type": "Windows",
    "managed_image_name": "{{user `Build_DefinitionName` | clean_image_name}}-{{isotime \"2006-01-02-1504\"}}-Build{{user `Build_BuildNumber`}}",
    "managed_image_resource_group_name": "{{user `wvd_goldimage_rg`}}",
    
    "image_publisher": "MicrosoftWindowsDesktop",
    "image_offer": "Windows-10",
    "image_sku": "19h1-evd",
    "communicator": "winrm",
    "winrm_use_ssl": "true",
    "winrm_insecure": "true",
    "winrm_timeout": "3m",
    "winrm_username": "packer",

    "managed_image_storage_account_type": "Premium_LRS",
    "temp_resource_group_name": "rg-PackerBuild-Prod-1",
    "virtual_network_name": "VNET-PROD-1",
    "virtual_network_subnet_name": "Subnet-PackerImage-Prod-1",
    "private_virtual_network_with_public_ip": "True",
    "virtual_network_resource_group_name": "rg-VNET-Prod-1",
    "azure_tags": {
        "Project": "Packer IT Image"
    },
    
    "location": "UK South",
    "vm_size": "Standard_B2S",

    "async_resourcegroup_delete":true
  }
],
```

**Explanation:**
Yes, I've been a bit naughty and **hard-coded** some of the property values. This is because I know that the production VNET name is unlikely going to change, for example.

You will notice that in this configuration the Packer VM is deployed into an **existing production VNET**: `"private_virtual_network_with_public_ip": "True",`. You can of course use a dedicated, isolated VNET for the Packer build run (default) but this comes with it's own challenges. In the environment this was run against, **Azure Policies** prevented VMs from being deployed to different VNETS. There are also **Storage Account firewalls** enabled, allowing only access from certain subnets/VNETS in the subscription.

Lets have a look at some of these new builder properties. A lot of these are self-explantory, so I won't cover those obvious ones:

* `"managed_image_name"` Automatically generated Azure VM Image object name as seen in the Azure portal. This is based on the Build number in Azure Devops, amongst other things.
* `"image_sku": "19h1-evd",` The **"evd"** is the Enterprise Virtual Desktop edition of Windows 10 required for Windows Virtual Desktop. You can use this guide from Microsoft to find different sku's, like the one that includes Office 365 `1903-evd-o365pp` [Find Windows VM images in the Azure Marketplace with Azure PowerShell](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/cli-ps-findimage)
* `"temp_resource_group_name": "rg-PackerBuild-Prod-1",` This RG **doesn't** need to exist. The Packer build will create it using the AAD Service Principal. It will get deleted again when the build completes (success or failure).
* `"virtual_network_subnet_name"` The subnet within the VNET that the temporary RG will be created in. Make sure it has an NSG that allows inbound WinRM 5986
* `"vm_size": "Standard_B2S",` is only used during the creation of the VM image - it has no bearing on what spec the WVD machines will be when deployed from said image.

At this stage we could actually run an Azure Packger build and it should sysprep and output an Image for us, but there are some further tweaks required first and we need at least a few applications installed in our Gold image from the Azure Files Share.

#### Packer PowerShell Provisioner

[Packer Provisioners](https://www.packer.io/docs/provisioners/index.html) _"use builtin and third-party software to install and configure the machine image after booting"_. So after the `builders` stage is finished, we can start to run our **PowerShell** code before restarting and then **sysprep'ing**.

In my PowerShell Provisioners I want to **map a drive** to the **Azure File Share** (created earlier) so that all the software for the Gold Image can be installed over SMB.

Installing [Chocolatey](https://chocolatey.org/) is also a good idea as many of the common packages can be installed directly from the public **Choco** repo.

**Edit** the Packer template again (**packer-win10_1903.json**) and **add two** new `powershell` Provisioners above the existing sysprep section, so the `provisioners` should now look like this:

```json
  "provisioners": [
    {
      "type": "powershell",
      "inline": [
          "$ErrorActionPreference='Stop'",

          "Invoke-Expression ((New-Object -TypeName net.webclient).DownloadString('https://chocolatey.org/install.ps1'))",
          "& choco feature enable -n allowGlobalConfirmation",
          "Write-Host \"Chocolatey Installed.\"",
      ]
    },
    {
      "type": "powershell",
      "inline": [
          "$ErrorActionPreference='Stop'",

          "Import-Module -Name Smbshare -Force -Scope Local",
          "$Usr='AzureAD\\'+\"{{user `AppInstallsStorageAccountName`}}\"",
          "New-SmbMapping -LocalPath J: -RemotePath \"{{user `packaged_app_installs_path`}}\" -Username \"$Usr\" -Password \"{{user `AppInstallsStorageAccountKey1`}}\"",
          "Write-Host \"'J:' drive mapped\"",

          "Set-MpPreference -DisableRealtimeMonitoring $true",
          "Write-Host \"Defender RealTime scanning temporarily disabled\"",

          "& \"J:\\Microsoft-PowerBIDesktop\\Install-PowerBIDesktop-Choco.ps1\"",
          "Write-Host \"'Microsoft Power BI Desktop' installed\"",

          "& \"J:\\FSLogix\\2.9.7117.27413\\x64\\Release\\Install-FSLogixAppsSetup.cmd\"",
          "Write-Host \"'FSLogix\\2.9.7117.27413' installed\""

      ]
    },
    {
      "type": "powershell",
      "inline": [
        " # NOTE: the following *3* lines are only needed if the you have installed the Guest Agent.",
        "  while ((Get-Service RdAgent).Status -ne 'Running') { Start-Sleep -s 5 }",
        "  while ((Get-Service WindowsAzureTelemetryService).Status -ne 'Running') { Start-Sleep -s 5 }",
        "  while ((Get-Service WindowsAzureGuestAgent).Status -ne 'Running') { Start-Sleep -s 5 }",

        "if( Test-Path $Env:SystemRoot\\windows\\system32\\Sysprep\\unattend.xml ){ rm $Env:SystemRoot\\windows\\system32\\Sysprep\\unattend.xml -Force}",
        "& $env:SystemRoot\\System32\\Sysprep\\Sysprep.exe /oobe /generalize /quiet /quit",
        "while($true) { $imageState = Get-ItemProperty HKLM:\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Setup\\State | Select ImageState; if($imageState.ImageState -ne 'IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE') { Write-Output $imageState.ImageState; Start-Sleep -s 10  } else { break } }"
      ]
    }
  ]

```

* The **first** of the `provisioners` installs **Choco** in it's own PowerShell thread, so that subsequent threads will load the new `%Path%` to **Choco.exe**.
* The **second** of the `provisioners` maps a `J:` drive to the **Azure File Share**, using the **Storage Account Name** and **key** which are provided by secure variables.
Soon we'll put the Build Pipeline together so that these values are passed into the Packer template.
  * I've also disabled realtime Defender scanning to speed up the build time.
  * There's also an example of installing **PowerBI Desktop** from the J: drive (which actually just kicks off a **Chocolatey** install)
  * and an **FSLogix Apps** install example.