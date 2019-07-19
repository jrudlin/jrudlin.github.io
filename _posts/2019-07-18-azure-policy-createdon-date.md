---
layout: post
title: CreatedOnDate tag for all resources in Azure using Azure Policy
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
  first_name: ''
  last_name: ''
---
Resources and objects in Azure don't contain a **created on date** property, this post will help you use Tags and Azure Policy to resolve that.

By using Azure Policy we will implement a **Policy** that applies to all resources, including Resource Groups, that automatically creates a **CreatedOnDate** Tag when the resource is created. 

## Date Created - Why?

_Why_ would you want to know when resources have been created in Azure?

Well for a start, **Azure Monitor audit logs are only kept by 90 days** so you can't rely on this to check historically.
The main reason would be for reporting. I wanted to run a **daily report** that produced all resources created on the previous day, **who** created them and how much the estimated **monthly cost** would be.

## Tags and Policy
Tags can be applied to almost any resource in Azure. Azure Policy is the service we will use to apply the tag: **CreatedOnDate** with.

Start by going to the [Policy](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyMenuBlade/Overview) blade in the Azure Portal:

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


The policy is also available here on my Github repo: [PolicyTagCreatedOnDate](https://github.com/jrudlin/Azure/blob/master/Policy/PolicyTagCreatedOnDate.json)

## What does it look like
When a resource is created in Azure, it will automatically be tagged with a **CreatedOnDate** and a **UTC** format date/time:

![CreatedOnDate]({{ site.baseurl }}/assets/images/Azure-Tag-CreatedOnDate-1.png)

## How can I use it?


