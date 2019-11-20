---
layout: post
title: Azure storage options for FSLogix profiles on Windows Virtual Desktop
subtitle: The different types of storage options when using FSLogix and WVD in Azure
date: 2019-08-17 20:39:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Windows 10
- Azure
- FSLogix
- Storage
- WVD
tags:
- Windows 10
- Windows Virtual Desktop
- FSLogix
- Win10
- Azure File Share
- Azure SMB Storage
- Azure Blob Storage
- Azure Costs
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

# FSLogix

FSlogix is essentially a roaming profile solution for Windows based on VHD(x) files.

With the advent of Windows Virtual Desktop (WVD) I needed to checkout which storage solution would best suit the needs of a Citrix deco project I was working on.

In order of preference, these are the Azure based storage options for FSLogix profiles that I identified:

## Storage Options
### 1. Azure File Share (SMB) - Computer Auth

 * Create an [Azure Premium File Share](https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-create-premium-fileshare)
 * Get the [cmdkey to access the file share](https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-windows)
 * Add the above storage credentials to the Windows Credential Manager as the SYSTEM account:
   * Run a startup script in Group Policy against the WVD VMs:
     `cmdkey /add:<storageaccountname>.file.core.windows.net /user:<storageaccountname> /pass:<storagekey>`
 * Set the FSLogix Registry value `AccessNetworkAsComputerObject` to `1`
 * Set the FSLogix Registry value `VHDLocations` to the UNC path of the Azure File Share

This is the most cost effective solution with excellent performance. It is also the simplest and quickest to setup and requires the least amount of infrastructure.

### 2. Azure Blob Storage (Cloud Cache)

 * Create an [Azure Premium v2 Storage Account](https://docs.microsoft.com/en-gb/azure/storage/common/storage-account-overview)
 * Copy the connection string for the storage account
 * Set the FSLogix Registry value `CCDLocations` with the connection string

Minimal infrastructure and good performance but beware of the costs. Premium Storage Accounts are charged per transaction!

### 3. Azure VM with Shared Folder (SMB)

 * [Create a profile container for a host pool using a file share](https://docs.microsoft.com/en-us/azure/virtual-desktop/create-host-pools-user-profile)
 * Set the FSLogix Registry value `VHDLocations` to the UNC path of the VM File Share

Decent performance (iops depend on VM size) but adds a single point of failure in the VM (unless you use Storage Spaces Direct S2D) which adds a lot of cost and complexity. Expensive.

### 4. Azure File Share (SMB) - User Auth

 * This is similar to option 1, except using Azure Active Directory Domain Services to authenticate the users access to the File Share
 * The VMs need to be joined to the Azure ADDS
 * Set the FSLogix Registry value `AccessNetworkAsComputerObject` to `0`
 * Set the FSLogix Registry value `VHDLocations` to the UNC path of the Azure File Share

Implementing and joining the WVD VMs to a different domain (Azure ADDS) is not something that would fit well in most environments. Having another domain to manage policies in - urgh!

## Conclusion

There are many other ways to host FSLogix profiles of course. Without additional licensing, and with just an Azure subscription, all the above options are available by default and I found that the **Azure File Share (SMB) - Computer Auth** option was the most suitable due to simplicity, cost and performance.
