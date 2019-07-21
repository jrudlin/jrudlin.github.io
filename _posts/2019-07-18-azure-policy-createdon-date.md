---
layout: post
title: CreatedOnDate tag for all resources in Azure using Azure Policy
subtitle: Resources and objects in Azure don't contain a created on date property, this post will help you use Tags and Azure Policy to resolve that.
share-img: ({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-1.png)
date: 2019-07-18 20:47:55.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Azure
- Tags
- Policy
tags:
- Azure
- Tags
- Policy
- CreatedOnDate
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: 'Jack'
  last_name: 'Rudlin'
---

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-1.png)

By using Azure Policy we will implement a **Policy** that applies to all resources, including Resource Groups, that automatically creates a **CreatedOnDate** Tag when the resource is created.

## Date Created - Why?

_Why_ would you want to know when resources have been created in Azure?

Well for a start, **Azure Monitor audit logs are only kept by 90 days** so you can't rely on this to check historically.
The main reason would be for reporting. I wanted to run a **daily report** that produced all resources created on the previous day, **who** created them and how much the estimated **monthly cost** would be.

## Tags and Policy

Tags can be applied to almost any resource in Azure. Azure Policy is the service we will use to apply the tag: **CreatedOnDate** with.

Start by going to the [Policy](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyMenuBlade/Overview) blade in the Azure Portal:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-2.png)

Click on **Definitions** and then **+ Poiicy definition** to create a new Policy:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-3.png)

In the **New Policy Definition** blade, enter a location at which the Policy will be applied, I went for the Management Group level so that all subscriptions I manage receive this poliicy - **Note:** it's best practice to create a **Management Group** under the Tenant Root and apply to that.
You can also apply to individual subscriptions.

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-4.png)

Give the new Policy a name:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-5.png)

Enter a description:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-6.png)

Create a new category for custom policies, or use an existing category. I went for existing. Categories allow for simplified filtering.

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-7.png)

Copy and paste the below JSON into the **Policy Rule**, delete anything that is already there first:

The latest CreatedOnDate policy is also available here on my Github repo: [PolicyTagCreatedOnDate](https://github.com/jrudlin/Azure/blob/master/Policy/PolicyTagCreatedOnDate.json)

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "tags['CreatedOnDate']",
          "exists": "false"
        }
      ]
    },
    "then": {
      "effect": "append",
      "details": [
        {
          "field": "tags['CreatedOnDate']",
          "value": "[utcNow()]"
        }
      ]
    }
  },
  "parameters": {}
}
```

It should look something like this:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-8.png)

Click on Save - at this point the policy is created but not assigned, so don't worry there is no chance of it having any impact at all yet.

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-9.png)

Back in the **Definitions** blade, find and click on your new policy by filtering on **Type > Custom**:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-10.png)

**Assign** the policy:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-11.png)

The Policy can be assigned to resources at any level **under** the original scope (which was the top level Management Group under the Tenant Root). You can also filter resources if there are specific ones you don't want the policy to apply to.
You can change the **Assignment name** if you so wish.
We don't need to deploy any resources to we don't need a **Managed Identity**

Once you click on **Assign**, any subscriptions in scope will after a few minutes, adhere to the new policy.

_Existing resources_ will not get the **CreatedOnDate** tag unless they are _updated/modified_, so I went through and add a  **CreatedOnDate** tag for all existing resources (using Az PowerShell) before I applied this policy. I used a value of **Existing** so I could determine they are older resources and therefore didn't know when they were created.

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-12.png)

## What does it look like

When a resource is created in Azure, it will automatically be tagged with a **CreatedOnDate** and a **UTC** format date/time:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-1.png)

## How can I use it?

Simple method, the Az PowerShell Module:

```powershell
Install-Module -Name Az -AllowClobber
Connect-AzAccount
$TagName = "CreatedOnDate"
Get-AzResource -TagName $TagName
```

Slightly more complex, using an Azure Automation runbook with a SendGrid email account to produce a daily report of all new resources and their costs:
[Get-RecentAzResource.ps1](https://github.com/jrudlin/Azure/blob/master/General/Get-RecentAzResource.ps1)