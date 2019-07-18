---
layout: post
title: Custom configuration of RD Web Pages
date: 2017-06-06 11:52:26.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- RDS
tags:
- RDS Remote Desktop RDWeb
meta:
  _publicize_job_id: '5830354598'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2017/06/06/first-blog-post/"
---
# Overview

Customising the look and feel of Remote Desktop Web Access pages can provide the following:

- Custom links
- Branding
- Intuitive interface
- Reduced clutter

The RD Web pages are stored on each of the RD Web Access servers in the system drive location:

- C:\Windows\Web\RDWeb

# Custom Windows 10 help link on the Login page

```powershell
#Set your RD Web servers
$RDBrokerWebGatewayServer1 = "rds-001"
$RDBrokerWebGatewayServer2 = "rds-002"
 
#Change to local help instead of out-dated 2008 R2 RDS help online
Invoke-Command -ComputerName $RDBrokerWebGatewayServer1,$RDBrokerWebGatewayServer2 -ScriptBlock {Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST/Default Web Site/RDWeb/Pages' -filter "appSettings/add[@key='LocalHelp']" -name "value" -value "true"}
```

Make the highlighted updates:

C:\Windows\Web\RDWeb\Pages\Site.xsl:

```html
<td class="cellSecondaryNavigationBar" height="40">
 <xsl:comment>Login Page only contains Help link</xsl:comment>
<table border="0" cellpadding="0" cellspacing="0" class="linkSecondaryNavigiationBarHelp">
<tr>
<td>
  <a id='PORTAL_HELP' href="javascript:onClickHelp()">
    <xsl:value-of select="$strings[@id = 'HelpLogin']"/>
  </a></td>
<td width="30"></td>
</tr>
```

In the **C:\Windows\Web\RDWeb\Pages\en-US\tswa.css** file add a new section:

```css
.linkSecondaryNavigiationBarHelp a
{
  color: #C00000;
  text-decoration: none;
  font-size: 14px;
  font-weight: bold;
}
```

In the **C:\Windows\Web\RDWeb\Pages\en-US\RDWAStrings.xml** file add a new section:

```html
<string id="HelpLogin">Using Windows 10? Click here for Help</string>
```

Result:

![RDCustom1]({{ site.baseurl }}/assets/images/rdcustom1.jpg)

## Changing the default rap-help.htm page

In the **C:\Windows\Web\RDWeb\Pages\en-US\rap-help.htm** file, edit section:

```html
<ul>
	<li id=WINDOWS10_1607><a href="#UserTopic_Windows_1607">Windows 10 1607 Remote Desktop fix</a></li>
	<li id=REMOTE_APP_DESKTOP_CONNECTIONS><a href="#UserTopic_What_is_RemoteApp_and_Desktop_Connections">What is RemoteApp and Desktop Connections?</a></li>
	<li id=REMOTE_PROGRAMS><a href="#UserTopic_What_are_Remote_Programs">What is RemoteApp?</a></li>
	<li id=STARTING_REMOTE_PROGRAMS><a href="#UserTopic_Starting_a_RemoteProgram">Starting a RemoteApp program</a></li>
	<li id=STARTING_REMOTE_DESKTOP_TAB><a href="#UserTopic_What_is_the_Remote_Desktop_tab">What is the Remote Desktop tab?</a></li>
	<li id=PUBLIC_PRIVATE_MODE><a href="#UserTopic_Public_Private_Mode">Public vs. private computer settings</a></li>
	<li id=COMPUTER_REQ><a href="#UserTopic_Computer_requirements">Computer requirements</a></li>
	<li id=CONTROL><a href="#UserTopic_Get_ActiveX">I am prompted to run the Remote Desktop Services ActiveX Client control. How do I do that?</a></li>
</ul>
```

In the same file, add a new section:

```html
<hr>

<h3><a name="UserTopic_Windows_1607"><id id=WINDOWS10_16072>Windows 10 1607 Remote Desktop fix</id></a></h3>
There is a known issue with Windows 10 version 1607.
    Type 'Winver' in start menu and press return to see your Windows 10 version.

Add the following registry value which you can do without administrator rights:

HKEY_CURRENT_USER\Software\Microsoft\Terminal Server Client
Name: RDGClientTransport
Type: Dword
Data: 1

For more information about this issue, see
    <a href="https://social.technet.microsoft.com/Forums/windowsserver/en-US/58521677-b54c-4285-9a06-9a966a9d8549/clean-install-windows-10-can-not-rdp-via-2012-rd2-rdg?forum=winserverTS" target="_blank">https://social.technet.microsoft.com</a>.

<a href="#_TopOfPage">Back to topics</a>
```

Result:

![RDCustom2]({{ site.baseurl }}/assets/images/rdcustom2.jpg)
![RDCustom3]({{ site.baseurl }}/assets/images/rdcustom3.jpg)