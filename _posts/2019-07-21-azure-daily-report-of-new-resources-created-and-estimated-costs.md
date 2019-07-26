---
layout: post
title: Daily email report of all new Azure resources created, their owner/creator, and estimated cost.
subtitle: Management type email report (costs and user who created all new resources within a timeframe) using Azure Automation runbooks, PowerShell and SendGrid.
share-img: "assets/images/Azure-Tag-CreatedOnDate-1.png"
date: 2019-07-21 20:47:55.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Azure
- Automation
- PowerShell
- SendGrid
tags:
- Azure
- Tags
- Policy
- CreatedOnDate
- Costs
- Script
- PowerShell
- Email
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![CreatedOnDate]({{ site.baseurl }}/assets/images/RecentAzResouce1.png)

Ever wanted a simple breakdown report/email of all new Azure resources/objects created within a fixed timeframe? Want to know who created the resource and how much it might cost for the month? Then read on....

## Overview

When senior management, the ones paying the Azure pay-as-you-go costs each month, suddenly want to know why they're spending **10k a month** on a high-spec SQL VM, you'll soon find that _who_ created this and _when_ is difficult information to retrieve from Azure....especially if the resource was created more than 90 days ago.

Objects and resources created in Azure don't have a "created date" property (for details on how to resolve this see previous post: [CreatedOnDate tag for all resources in Azure using Azure Policy](/2019-07-18-azure-policy-createdon-date)). They also don't have a "created by" user property, but this info can be scraped from Azure Monitor logs within 90 days.

Combining these two properties and adding an estimated cost, we can keep management happy knowing that resources created were pre-approved and that costs have been justified.

## Setup the daily report
### SendGrid
Create a SendGrid account in Azure, if you don't already have one. I'm using the free tier of SendGrid in my Azure subscription. There are some limitations around advanced security, but for the purposes of a single daily email it suffices.

![SendGrid]({{ site.baseurl }}/assets/images/RecentAzResouce5.png)

![SendGrid]({{ site.baseurl }}/assets/images/RecentAzResouce6.png)

### Azure Automation Runbook - Email-SendGrid
I'm using a Runbook for SendGrid that can be re-used by providing parameters from another Runbook, so it's not limited to just this solution.

Retrieve the username from your SendGrid account:
![CreatedOnDate]({{ site.baseurl }}/assets/images/RecentAzResouce7.png)

Use this username, and the password you set when creating the SendGrid account, to create credentials in the Azure Automation account:
![CreatedOnDate]({{ site.baseurl }}/assets/images/RecentAzResouce3.png)
![CreatedOnDate]({{ site.baseurl }}/assets/images/RecentAzResouce4.png)

Create a new Runbook in your Automation Account, called Email-SendGrid
![CreatedOnDate]({{ site.baseurl }}/assets/images/RecentAzResouce2.png)

Paste the code from [Email-SendGrid.ps1](https://github.com/jrudlin/Azure/blob/master/General/Email-SendGrid.ps1) into this new Runbook.

Modify the following variables in the script:

```powershell
# The name of you SendGrid credentials stored in the Automation Account
$SendGridAdminAccount = "SendGridAlertsProd"

$EmailFrom = "AzureAlerts@jrudlin.org.uk"
```

### Azure Automation Runbook - Get-RecentAzResource
Modules you'll need to install in your Automation account:

- Az.Billing
- Az.Accounts
- Az.Automation
- Az.Monitor
- Az.Resources

As mentioned above, you need the [CreatedOnDate tag](/2019-07-18-azure-policy-createdon-date) Azure Policy in place first.

Grab a copy of the [Get-RecentAzResource.ps1](https://github.com/jrudlin/Azure/blob/master/General/Get-RecentAzResource.ps1) script and drop it into another new Automation Runbook.

The only mandatory variable changes would be the following

```powershell
# Your Automation credentials that have ReadOnly rights to all subscriptions
$AzROAccount = "AzReadOnlyAccount@domain.co.uk"

# Recipients to receive the report
$EmailRecipients = "Jack.Rudlin@domain.co.uk","Jack.Test@domain.org.uk"

# Automation account details and name of the runbook created for the SendGrid email
$AutomationAccount = 'Azure Automation Account Name'
$AutomationAccountRG = 'Azure Automation Account RG'
$Runbook = 'Email-SendGrid'
```

You will also need an Automation Account **AzureRunAsConnection** which only needs permissions to run Runbooks. Remember, when you create a RunAsAccount for the first time it will give itself Contributor rights on your subscription, so you should change this.
```powershell
$ServicePrincipalConnection = Get-AutomationConnection -Name 'AzureRunAsConnection'
```

Schedule your **Get-RecentAzResource** to run once a day in the evening.

'et voilà:
![CreatedOnDate]({{ site.baseurl }}/assets/images/RecentAzResouce1.png)

## The Script caveats
I am trying to estimate the costs based on resources being charged on a daily basis. If any resources are added and are charged montly (like Azure Devops), then the pricing may be inaccurate.

I'm grabbing info from lots of different services in Azure:

- Azure Monitor Logs are providing the username who created the resource.
- Azure Resources is getting the CreatedOnDate tag and the resource details.
- Azure Billing is providing the cost of the resources.

Enjoy.