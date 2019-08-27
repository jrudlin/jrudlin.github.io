---
layout: post
title: Shrink an Azure VMs OS Managed Disk using PowerShell.
subtitle: Using PowerShell (and the Azure Portal) to reduce/shrink the OS Managed Disk size for a Windows VM in Azure.
share-img: "assets/images/intune-win10-login-script1.png"
date: 2019-08-27 20:48:00.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Windows
- Azure
- Storage
- PowerShell
- Virtual Machine
tags:
- Azure VM
- Managed Disks
- PowerShell
- Storage Account
- OS Disk
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![Existing Disk]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk1.png)

Using PowerShell (and the Azure Portal) to reduce/shrink the OS Managed Disk size for a Windows VM in Azure.

## Overview

At present it is not supported to reduce/shrink the OS disk (managed or unmanaged) size of an Azure VM from the Azure Portal (say from 128Gb to 32Gb for example), using PowerShell or any other tools. A Microsoft [blog post](https://devblogs.microsoft.com/premier-developer/how-to-shrink-a-managed-disk/) covers this statement and also an alternative approach to this post.

This is an adapted version of my original post [Shrink Azure VM OS disk size (reduce)](https://jrudlin.github.io/2017/10/31/resize-azure-vm-vhd-blob-to-smaller-disk-size-downsize/)

### Scenario

For **cost saving** (and performance) reasons you may want to reduce the size of the OS Disk (or data disks for that fact) that are already assigned to a running VM - **if there is sufficient space within the volume** to first shrink the volume within the OS.

Many on-prem VMs are built from a VMWare template, often with an oversized OS drive to cater for growth which never materialises. If you find there is plenty of free space to go down to a 64GB or even a 32GB OS Disk, then check out these steps.

### Alternative Approach

The technique used to the reduce the disk in this post is identical to that of the Microsoft post mentioned above, except that here **we use only PowerShell without relying on an opensource tool**. This process in this post is also **dynamic** in that it reads the footer from a new disk created in Azure, ensuring the **correct value is always written** and up to date.

## Shrink Disk

### Pre-Reqs

In this example, the Azure VM AAHW2 (_yeah I know you can see I've done a bad job of masking the first few chars of the original name_) has a managed OS disk size of **100Gb** running the Standard HDD storage type. As the disk is Managed, we will be charged for a **128GB** disk:
![Existing Disk]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk1.png)

We can see that there is sufficient space on the C drive to **reduce** the Azure Managed disk size to **32Gb**:
![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk2.png)

### Shrink the OS Partition

* Open PowerShell as Administrator
* Run: `Get-Partition -DiskNumber 0` where `0` is the Disk number of the OS Disk:
![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk3.png)

* We now have the PartitionNumber which is **"2"** – this is the OS Disk **"C"**

* Work out what the new OS Volume size will be **+** the existing System Reserved partition.
  
  So if we set the OS partition to **31GB** and add **500MB** for the System Reserved partition we get **31.5Gb** in total for Disk0. This is within the **32Gb** Azure Disk size so will work in this scenario.

* In PowerShell run: `Get-Partition -DiskNumber 0 -PartitionNumber 2 | Resize-Partition -Size 31GB`
Here we have specified the PartitionNumber retrieved above and the new OS Disk partition Size that we want (based on the Azure disk size of **32Gb**)

* You should end up with the below disk configuration:
![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk4.png)

* If you get the below error
> The specified shrink size is too big and will cause the volume to be smaller than the minimum volume size

  just run the PowerShell cmd a few more times until it succeeds:
![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk5.png)

* Shutdown the VM from within the OS
* Deallocate the VM from the Azure portal:
![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk6.png)

![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk7.png)

* Wait until the VM becomes Stopped (deallocated):
![Existing Volumes]({{ site.baseurl }}/assets/images/shrink-azure-vm-osdisk8.png)


### Resize the Azure VM OS Managed Disk

* Open PowerShell as Administrator and install the Az module:
Install-Module Az
