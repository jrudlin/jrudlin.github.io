---
layout: post
title: Domain name registration transfer to Azure App Service domains
date: 2018-10-27 07:47:02.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Azure
- PowerShell
tags:
- app service
- azure app service domain
- domain registration
- domain transfer
meta:
  _publicize_done_18837840: '1'
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _thumbnail_id: '282'
  _publicize_job_id: '23632879513'
  timeline_notification: '1540626424'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1056089887353397248";}}
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  publicize_linkedin_url: www.linkedin.com/updates?topic=6461855593517903872
  _publicize_done_18837845: '1'
  _wpas_done_18662064: '1'
  _oembed_3af2389f251e642d6592c72a9f79f186: "{{unknown}}"
  _oembed_9ba6aadb0feba7bdf713002be4fbd542: "{{unknown}}"
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/10/27/domain-name-registration-transfer-to-azure-app-service-domains/"
---
**Updated 03/11/2018** [A transfer-in of a .uk domain into Azure is not currently supported as the IPSTAG is required by Nominet on the existing provider side. I assume it would be GODADDY when transferring into Azure. The Azure portal will be updated soon to support UI based migrations]

# What?

If you have some domain names registered with say 123-reg or another provider and want to migrate/**transfer the ownership into Azure** , you can do this with the supported top level domains: **com** , **net** , **co.uk** , **org** , **nl** , **in** , **biz** , **org.uk** , and **co.in** (as documented here: [Buy a custom domain name for Azure Web Apps](https://docs.microsoft.com/en-us/azure/app-service/custom-dns-web-site-buydomains-web-app).

# Why?

Some of the reasons you might want to do this:

- Take advantage of Microsoft Azure's flat rate pricing, for all domains, that they have agreed with GoDaddy
- Single console to control domains, DNS, traffic manager, web sites (app service) etc. etc.
- Better automation/api functionality (in my opinion) than what some of the domain name hosting companies offer.

# How?

There are a few blogs on the internet on how to achieve this with PowerShell using:

```powershell
New-AzureRmResource -ResourceType Microsoft.DomainRegistration/domains  
```

like on [Jos Liebens](https://www.lieben.nu/liebensraum/2017/07/transferring-a-domain-to-azure-dns-and-billing/) site.

## Issue

However, like others had commented, I also received this error back after running appropriate PoSh:

```powershell
New-AzureRmResource : {"Code":"BadRequest","Message":"Parameter domain is null or empty.","Target":null,"Details":[{"Message":"Parameter domain is null or empty."},{"Code":"BadRequest"},{"ErrorEntity":{"ExtendedCode":"51011","MessageTemplate":"Parameter {0} is null or empty.","Parameters":["domain"],"Code":"BadRequest","Message":"Parameter domain is nullor empty."}}],"Innererror":null}  
```

So seems there maybe a bug with this AzureRM cmdlet? I couldn't see this domain property mentioned in the [Microsoft.DomainRegistration/domains](https://docs.microsoft.com/en-us/azure/templates/microsoft.domainregistration/domains) documentation.

## Solution

The Microsoft Azure [REST API](https://docs.microsoft.com/en-us/rest/api/azure/).

There are probably other ways to initiate a domain name transfer into Azure using the REST API, but I found this way to be pretty simple.

1. Go to the [Domains - Create Or Update](https://docs.microsoft.com/en-us/rest/api/appservice/domains/createorupdate) page where you interact with the API from the Microsoft docs page.

2. Click on the '**Try it**' button and login with your Azure AD credentials. ( I have global admin permissions in my tenant ).  
![Azure_TryIt]({{ site.baseurl }}/assets/images/azure_tryit.png)

3. Add the mandatory parameters:  
**resourceGroupName** - where the App Service object will be created  
**domainName** - the domain name you are migrating from another provider into Azure  
**api-version** - I left this as default ![domain_transfer_params]({{ site.baseurl }}/assets/images/domain_transfer_params.jpg)

4. For a domain transfer, I used the following body:  
**Note:** some of the properties are mandatory/required
```json
{  
  location: "Global",  
  properties: {  
   contactAdmin: "Jack Rudlin",  
   contactBilling: "Jack Rudlin",  
   contactRegistrant: "Jack Rudlin",  
   contactTech: "Jack Rudlin",  
   privacy: "True",  
   autoRenew: "True",  
   authCode: "q\\1u{b=wbY9bNT193iNS",  
   Consent: {  
    agreedAt: "2018-10-21T20:10:40",  
    agreedBy: "70.80.90.100",  
    agreementKeys: ["DNPA","DNTA"]  
   }  
  }
}  
```

![domain_transfer_body]({{ site.baseurl }}/assets/images/domain_transfer_body.jpg)

You should get a **202** response back if the post was successful

**Note: Don't forget to escape your JSON! Check the authCode. I had a backslash \ in mine so I had to escape it with an additional \

1. In the Azure Resource Group that you specified in the earlier parameters, the App Service should be listed with the domain name you are transferring: ![rg]({{ site.baseurl }}/assets/images/rg.png)

2. A day or two later, the annual charge for the domain hosting service should be taken from your Azure funds: ![azure domain cost]({{ site.baseurl }}/assets/images/azure-domain-cost.png)

3. Finally once the domain transfer has been successfully completed, you will get access to manage the domains DNS:  
 ![appdomain_active]({{ site.baseurl }}/assets/images/appdomain_active.png)

4. Post domain transfer you'll probably want to migrate your DNS and then web services.

I quite liked using the REST API post method from the browser. In an enterprise environment, I can immediately see these benefits:

- Browser supports authenticated proxies natively - PowerShell has issues with this
- No need to download/install modules for PowerShell
- No local administrator rights required
- I guess the Azure cloud shell is similar, but that requires a storage account and has an additional cost association
