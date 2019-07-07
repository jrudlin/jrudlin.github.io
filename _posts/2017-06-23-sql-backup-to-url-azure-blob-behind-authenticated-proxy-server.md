---
layout: post
title: SQL Backup to URL (Azure Blob) behind authenticated proxy server
date: 2017-06-23 19:35:47.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- SQL
tags:
- authenticated proxy
- Backup to URL
- BackuptoURL
- proxy server
- SQL 2012
- SQL 2014
- SQL 2016
meta:
  _rest_api_client_id: "-1"
  _rest_api_published: '1'
  _publicize_job_id: '6419548522'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2017/06/23/sql-backup-to-url-azure-blob-behind-authenticated-proxy-server/"
---
If you want to use SQL 2012 or above to backup directly to a URL and your servers are behind a corporate proxy that also requires auth, create the following .net config file so that the sql service will make the connection correctly:

c:\Program Files\Microsoft SQL Server\MSSQL11.MSSQLSERVER\MSSQL\Binn\ **BackuptoURL.exe.config**

Add the following code with your proxy address to the new file:

[code language="xml"]  
\<?xml version ="1.0"?\>  
\<configuration\>  
 \<system.net\>  
 \<defaultProxy\>  
 \<proxy proxyaddress="http://proxylb.domain.local:8080" bypassonlocal="true" usesystemdefault="True" /\>  
 \<bypasslist\>  
 \<add address="[a-z]+.domain.local$" /\>  
 \</bypasslist\>  
 \</defaultProxy\>  
 \</system.net\>  
\</configuration\>  
[/code]

