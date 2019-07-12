---
layout: post
title: SCCM Technical Preview 1808 - Repair app from Software Center
date: 2018-08-23 13:35:31.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- SCCM
tags:
- Application Deployment
- ConfigMgr
- SCCM Application Repair
- SCCM Technical Preview 1808
meta:
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1032622415707365376";}}
  timeline_notification: '1535031341'
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '21382932809'
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
permalink: "/2018/08/23/sccm-technical-preview-1808-repair-app-from-software-center/"
excerpt: 'A quick overview of the new "repair" application feature in SCCM 1808 Tech
  Preview. '
---
Quick post today. I just wanted to show you the new Repair Application feature from SCCM 1808's Software Center.

In the previous TP release, the repair option was available in the Deployment Type of the Application but was not exposed in the Software Center UI. Now in the new SCCM 1808 Technical Preview we can use this new option from the Software Center.

The agent version of the machine running the repair option from Software Center should be 5.0.0.8707.1000 at least. The site version should be 1808+ which is also┬á5.0.0.8707.1000.

So lets get started and see how this new feature is implemented from the SCCM Console and how the UX is from the users perspective in the Software Center.

1. Update your existing Application, or create a new one. Add a repair program in.  
  
**Note:** You can usually find the ModifyPath of a an existing installed app from the Registry:┬á **Computer\HKEY\_LOCAL\_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\**  
However, beware this key represents a 'modify', so it would probably be interactive and may not be suitable for your silent repair.

![SCCM Repair app 2]({{ site.baseurl }}/assets/images/sccm-repair-app-2.jpg)

2. Deploy your Application that has the repair option. You can only deploy to a User collection. Tick the box to " **Allow end users to attempt to repair this application**"

![SCCM Repair app 1]({{ site.baseurl }}/assets/images/sccm-repair-app-1.jpg)

3. Jump onto your client as the user who you deployed the app to and install the deployed app. Once installed you will then see the new Repair option available.

![SCCM Repair app 3]({{ site.baseurl }}/assets/images/sccm-repair-app-3.jpg)

4. Run the new Repair option to see it in action! The user will get a prompt with this warning:

![SCCM Repair app 4]({{ site.baseurl }}/assets/images/sccm-repair-app-4.jpg)

5. Once the repair has finished, we can see from the AppEnforce.log that the repair cmd has successfully run:

![SCCM Repair app 5]({{ site.baseurl }}/assets/images/sccm-repair-app-5.jpg)

6. Thanks, and enjoy. I told you it was a quick one
