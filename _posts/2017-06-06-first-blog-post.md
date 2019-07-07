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

[code language="powershell"]  
#Set your RD Web servers  
$RDBrokerWebGatewayServer1 = "rds-001"  
$RDBrokerWebGatewayServer2 = "rds-002"

#Change to local help instead of out-dated 2008 R2 RDS help online  
Invoke-Command -ComputerName $RDBrokerWebGatewayServer1,$RDBrokerWebGatewayServer2 -ScriptBlock {Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST/Default Web Site/RDWeb/Pages' -filter "appSettings/add[@key='LocalHelp']" -name "value" -value "true"}  
[/code]

Make the highlighted updates:

C:\Windows\Web\RDWeb\Pages\Site.xsl:

[code language="html" highlight="3,7"]  
\<td class="cellSecondaryNavigationBar" height="40"\>  
 \<xsl:comment\>Login Page only contains Help link\</xsl:comment\>  
\<table border="0" cellpadding="0" cellspacing="0" class="linkSecondaryNavigiationBarHelp"\>  
\<tr\>  
\<td\>  
 \<a id='PORTAL\_HELP' href="javascript:onClickHelp()"\>  
 \<xsl:value-of select="$strings[@id = 'HelpLogin']"/\>  
 \</a\>\</td\>  
\<td width="30"\>\</td\>  
\</tr\>  
[/code]

C:\Windows\Web\RDWeb\Pages\en-US\tswa.css  
Add a new section:

[code language="html" highlight="1,2,3,4,5,6,7"]  
.linkSecondaryNavigiationBarHelp a  
{  
 color: #C00000;  
 text-decoration: none;  
 font-size: 14px;  
 font-weight: bold;  
}  
[/code]

C:\Windows\Web\RDWeb\Pages\en-US\RDWAStrings.xml  
Add a new section:

[code language="html" highlight="1"]  
\<string id="HelpLogin"\>Using Windows 10? Click here for Help\</string\>  
[/code]

Result:

![RDCustom1]({{ site.baseurl }}/assets/images/rdcustom1-e1496761527670.jpg?w=300)

## Changing the default rap-help.htm page

C:\Windows\Web\RDWeb\Pages\en-US\rap-help.htm  
Edit section:

[code language="html" highlight="2"]  
\<ul\>  
 \<li id=WINDOWS10\_1607\>\<a href="#UserTopic\_Windows\_1607"\>Windows 10 1607 Remote Desktop fix\</a\>\</li\>  
 \<li id=REMOTE\_APP\_DESKTOP\_CONNECTIONS\>\<a href="#UserTopic\_What\_is\_RemoteApp\_and\_Desktop\_Connections"\>What is RemoteApp and Desktop Connections?\</a\>\</li\>  
 \<li id=REMOTE\_PROGRAMS\>\<a href="#UserTopic\_What\_are\_Remote\_Programs"\>What is RemoteApp?\</a\>\</li\>  
 \<li id=STARTING\_REMOTE\_PROGRAMS\>\<a href="#UserTopic\_Starting\_a\_RemoteProgram"\>Starting a RemoteApp program\</a\>\</li\>  
 \<li id=STARTING\_REMOTE\_DESKTOP\_TAB\>\<a href="#UserTopic\_What\_is\_the\_Remote\_Desktop\_tab"\>What is the Remote Desktop tab?\</a\>\</li\>  
 \<li id=PUBLIC\_PRIVATE\_MODE\>\<a href="#UserTopic\_Public\_Private\_Mode"\>Public vs. private computer settings\</a\>\</li\>  
 \<li id=COMPUTER\_REQ\>\<a href="#UserTopic\_Computer\_requirements"\>Computer requirements\</a\>\</li\>  
 \<li id=CONTROL\>\<a href="#UserTopic\_Get\_ActiveX"\>I am prompted to run the Remote Desktop Services ActiveX Client control. How do I do that?\</a\>\</li\>  
\</ul\>  
[/code]

Add a new section:

[code language="html"]

\<hr\>

\<h3\>\<a name="UserTopic\_Windows\_1607"\>\<id id=WINDOWS10\_16072\>Windows 10 1607 Remote Desktop fix\</id\>\</a\>\</h3\>  
There is a known issue with Windows 10 version 1607.  
 Type 'Winver' in start menu and press return to see your Windows 10 version.

Add the following registry value which you can do without administrator rights:

HKEY\_CURRENT\_USER\Software\Microsoft\Terminal Server Client  
Name: RDGClientTransport  
Type: Dword  
Data: 1

For more information about this issue, see  
 \<a href="https://social.technet.microsoft.com/Forums/windowsserver/en-US/58521677-b54c-4285-9a06-9a966a9d8549/clean-install-windows-10-can-not-rdp-via-2012-rd2-rdg?forum=winserverTS" target="\_blank"\>https://social.technet.microsoft.com\</a\>.

\<a href="#\_TopOfPage"\>Back to topics\</a\>

[/code]

Result:

![RDCustom2]({{ site.baseurl }}/assets/images/rdcustom2.jpg) ![RDCustom3]({{ site.baseurl }}/assets/images/rdcustom3.jpg)

