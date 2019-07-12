---
layout: post
title: Lenovo graphics driver for P51s and ThinInstaller detection issue
date: 2018-04-28 14:14:26.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Drivers
- Lenovo
- SCCM
- ThinInstaller
- Update Retriever
tags:
- Intel HD Graphics 620
- Lenovo ThinInstaller
- Lenovo Update Retriever
- P51s drivers
meta:
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '17278258228'
  timeline_notification: '1524924868'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:56:"https://twitter.com/JackRudlin/status/990232786903687168";}}
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  _publicize_failed_18837845: O:13:"Keyring_Error":2:{s:6:"errors";a:1:{s:21:"keyring-request-error";a:1:{i:0;a:6:{s:7:"headers";O:42:"Requests_Utility_CaseInsensitiveDictionary":1:{s:7:"
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2018/04/28/lenovo-graphics-driver-for-p51s-and-thininstaller-detection-issue/"
---
Oh I spend so much time troubleshooting Lenovo's UpdateRetriever issues these days. Seems Lenovo really don't test much at all. In the latest Win10 x64 Graphics driver release for the Lenovo ThinkPad P51s, the driver is no longer detected and installed.

I run ThinInstaller during an SCCM Task Sequence so only noticed this post build - the Microsoft default display driver was installed instead of the Intel HD 620.

The faulty Lenovo driver package is:┬á **n1ndt16w\_10**

The detection failure happens on an Lenovo ThinkPad P51s System Model **20HC** S12D00

Intel HD Graphics 620 version:┬á **23.20.16.4905**

Missing Hardware ID: **PCI\VEN\_8086&DEV\_5916&SUBSYS\_224817AA**

Open the┬á **n1ndt16w\_10** package in Update Retriever:

![UpdateRetriever1]({{ site.baseurl }}/assets/images/updateretriever1.jpg)

On the┬á **Define Dependencies** tab, add a new PNP ID:

![UpdateRetriever2]({{ site.baseurl }}/assets/images/updateretriever2.jpg)

When ThinInstaller runs again, it will be able to detect the Intel HD 620 is installed in the laptop and run this package installer.

I found this issue by enabling debug in the┬á **ThinInstaller.exe.configuration** file located in the ThinInstaller directory. I saw that none of the PNP ID's matched.

