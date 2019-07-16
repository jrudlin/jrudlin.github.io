---
layout: post
title: SCCM Data Warehouse - SQL SSL certificates on AlwaysOn cluster
date: 2018-02-13 13:34:28.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- PowerShell
- SCCM
- SQL
tags:
- Data Warehouse
- SQL 2016
- SQL Server
meta:
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '14703320324'
  timeline_notification: '1518528869'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:56:"https://twitter.com/JackRudlin/status/963406026027479041";}}
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
permalink: "/2018/02/13/sccm-data-warehouse-sql-ssl-certificates-on-alwayson-cluster/"
---
I was setting up the new Data Warehouse feature of SCCM on our corporate SQL 2016 AlwaysOn cluster.

You will notice that after you have installed the DW and try to run a report, you may get this error:

> An error has occurred during report processing. (rsProcessingAborted) Cannot create a connection to data source ‘AutoGen__39B693BB_524B_47DF_9FDB_9000C3118E82_’. (rsErrorOpeningConnection) A connection was successfully established with the server, but then an error occurred during the pre-login handshake. (provider: SSL Provider, error: 0 – The certificate chain was issued by an authority that is not trusted.)

The Data Warehouse reporting data source has this parameter:
> TrustServerCertificate=false

You can change this to true - but it will get set back to false by SCCM :)

The correct way forward - in my my opinion, is to install trusted certificates from your root CA on the SQL Instances hosting the DW database. There are other posts on installing the self signed certificate as a trusted root CA onto the reporting server, but I don't like using self signed certs as they are difficult to track and revoke.

This solution was tested on a 3 node AlwaysOn SQL 2016 cluster running on Server 2016 VMs in Azure.

The certificates are requested from the internal publishing CA's using powershell.

The SQL **instance** where the Data Warehouse database exists will likely have \>1 Availability Group and Availability Group Listener (AGL), therefore when requesting the certificate, ALL the AGLs netbios and fqdn's should be included in the Subject Alternative Name (SAN) properties, as well as the hostname.

I ran this PowerShell on each of the 3 SQL nodes to request a unique certificate.

```powershell
# Create and submit a request  
$Hostname = [System.Net.Dns]::GetHostByName(($env:computerName)).Hostname  
$template = "WebServer2008"  
$SAN = $Hostname,"AGL1-INSTANCE1","AGL1-INSTANCE1.domain.local","AGL2-INSTANCE1","AGL2-INSTANCE1.domain.local","AGL3-INSTANCE1","AGL3-INSTANCE1.domain.local","AGL4-INSTANCE1","AGL4-INSTANCE1.domain.local"  
Get-Certificate -Template $template -DnsName $SAN -CertStoreLocation cert:\LocalMachine\My
```

Now, hop onto your issuing CA and approve the pending request for the certificate.

Back on the SQL nodes, retrieve the requested certificate from the CA. It will be installed into the machines Personal store.

```powershell
# Now the certificate is approved, you can retrieve the request:  
Get-Certificate -Request (Get-ChildItem -Path cert:\LocalMachine\Request)  
```

Open an MMC and add the Certificates snap-in targeted at the local computer store. Find the newly installed certificate (The one with the above SANs in it) and retrieve the certificate thumbprint.

On each of the SQL Nodes, change the following registry value so that it matches the thumbprint of the newly approved certificate:

```windows registry entries
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL13.INSTANCE1\MSSQLServer\SuperSocketNetLib]  
"Certificate"="25615102f32108599d89555678e7fcdabb0db491"
```

The SQL Server service for the instance will need restarting following this Registry update.

Before testing the SCCM Data Warehouse reports, you can try in SQL Server Management Studio using these options:

![SQLServerSSL]({{ site.baseurl }}/assets/images/sqlserverssl.png)

