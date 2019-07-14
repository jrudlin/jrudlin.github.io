---
layout: post
title: Simple SQL Maintenance for your SCCM / ConfigMgr environment - PowerShell CI
date: 2019-01-13 20:13:08.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Automation
- PowerShell
- SCCM
- SQL
tags:
- Configuration baseline
- Configuration Item
- Ola Hallengren SCCM
- SCCM ConfigMgr SQL Maintenance
- SQL Maintenance Compliance
meta:
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1084543898821844992";}}
  _publicize_job_id: '26496326000'
  timeline_notification: '1547410389'
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  publicize_linkedin_url: www.linkedin.com/updates?topic=6490309593510531072
  _publicize_done_18837845: '1'
  _wpas_done_18662064: '1'
  publicize_google_plus_url: https://plus.google.com/101508538917585204483/posts/1w3RktsMMCK
  _publicize_done_21144400: '1'
  _wpas_done_21561694: '1'
  _thumbnail_id: '325'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2019/01/13/simple-sql-maintenance-for-your-sccm-configmgr-environment-powershell-ci/"
---

I took this idea from one of my previous posts [Simple WSUS Maintenance/Cleanup for your SCCM environment](/2018/11/06/simple-wsus-maintenance-cleanup-for-your-sccm-environment) where a PowerShell script can be loaded into an **SCCM Configuration Item** and run automagically across all the SCCM servers using a dynamic collection. Hands off and completely automated.

This time round it's the renowned **Ola Hallengren** [SQL maintenance](https://ola.hallengren.com/sql-server-index-and-statistics-maintenance.html) scripts. Ola's scripts have always been recommended by ConfigMgr/SCCM MVPs and partners etc. The re-indexing provides performance gains in the CM database. SQL databases need maintenance as over time, the data/tables/indexes can become fragmented and this leads to poor performance.

You can check for SQL fragmentation using this T-SQL:

```sql
sp_helpdb
```

This will give you a list of the databases and mostly importantly, the **dbid.&nbsp;** The&nbsp;ConfigMgr&nbsp;database&nbsp;in&nbsp;this&nbsp;example&nbsp;has&nbsp;ID&nbsp;=&nbsp;7

![sp_helper](/assets/images/sp_helpdb.jpg)

```sql
select * from sys.dm_db_index_physical_stats (7,DEFAULT,DEFAULT,DEFAULT,DEFAULT)
where page_count > 1000
order by avg_fragmentation_in_percent desc 
```

This will give you the **avg\_fragmentation\_in\_percent** column. You can use this to compare before/after running the Ola Hallengren maintenance scripts.

In an SCCM hierarchy, each CAS and Primary Site will need its own database. These are generally run on a single SQL server per SCCM site. Software Update Points also use SQL databases and can sometimes share the SQL Server with the SCCM Primary Site Server. SUPs (WSUS) can also share databases.

I have developed a simple [PowerShell script](https://github.com/jrudlin/SCCM-SQL-Maintenance/blob/master/SCCM%20SQL%20Ola%20Hallengren%20Maintenance%20Scripts.ps1) that uses an SCCM Configuration Item (CI) /Baseline to deploy the [Ola Hallengren](https://ola.hallengren.com/) SQL maintenance onto the SCCM/SUP SQL Servers and maintain the SQL Agent scheduling for the various maintenance tasks. It is effectively a DSC/Compliance for the Agent Jobs/Schedules.

The CI is targetted at a dynamic collection containing all SCCM SQL servers and SUPs.


**Assumptions:**

- The SYSTEM account on each of the SQL / WSUS servers has sysadmin access to the SQL server
- SQL is installed locally on the WSUS/SUP server (if this is not the case, just change the collection query appropriately)


# Script Overview

The [script](https://github.com/jrudlin/SCCM-SQL-Maintenance/blob/master/SCCM%20SQL%20Ola%20Hallengren%20Maintenance%20Scripts.ps1)does the following:

- Retrieves the Ola Hallengren **MaintenanceSolution.sql** from the disk or network
- Loads the SQL Server PoSh module from the disk or network
- Writes to a local CMTrace formatted log file
- Installs **MaintenanceSolution.sql** into the Master database
- Creates SQL Agents Jobs
- Changes the SQL Agent service startup to automatic
- Sets SQL Agent Job schedules
- Upgrades the **MaintenanceSolution.sql** scripts if it detects a new version available on the disk or network
- Supports adding custom TSQL to the job steps
- Tested with SCCM 1802+ and SQL 2012+


## Installation

### Pre-reqs

- Disable all existing SQL maintenance jobs but not backup jobs.  
Disable Jobs that are related to: DBCC/Integrity Checks/Indexing 
- Download a copy of my [SCCM SQL Ola Hallengren Maintenance Scripts.ps1](https://github.com/jrudlin/SCCM-SQL-Maintenance/blob/master/SCCM%20SQL%20Ola%20Hallengren%20Maintenance%20Scripts.ps1). 
- Download the latest copy of [MaintenanceSolution.sql](https://github.com/olahallengren/sql-server-maintenance-solution/blob/master/MaintenanceSolution.sql) 
- Copy the [MaintenanceSolution.sql](https://github.com/olahallengren/sql-server-maintenance-solution/blob/master/MaintenanceSolution.sql) file to a network share that the computer accounts of the SCCM SQL Server and SUP Servers can access - this is the recommended approach so that all SQL servers can access the same version. 
- Grab a copy of the [SQLServer](https://www.powershellgallery.com/packages/SqlServer)module from the PowerShell gallery and copy to a network share. Same permissions as mentioned above.


### Script Preparation

In the downloaded copy of [SCCM SQL Ola Hallengren Maintenance Scripts.ps1](https://github.com/jrudlin/SCCM-SQL-Maintenance/blob/master/SCCM%20SQL%20Ola%20Hallengren%20Maintenance%20Scripts.ps1), amend **line 32** variable:


```PowerShell
$OlaHallengrenScriptLocation =
```

so that it reflects  
the UNC path of the [MaintenanceSolution.sql](https://github.com/olahallengren/sql-server-maintenance-solution/blob/master/MaintenanceSolution.sql)  
file.

Also confirm that **line 37** variable:

```PowerShell
$SQLServerPoShModule =
```

points to a location that contains the SQLServer module file **SqlServer.psm1** and that the location is also accessible by the computer accounts of the SCCM SQL Server and SUP Servers.

Adjust the schedules to your liking:

```powershell
$SQLJobsConfig
```

This variable contains all the SQL Agent jobs you want to configure. In my script they are mostly the Ola Hallengren ones. You can add your own custom jobs if they need configuring.

**Note:&nbsp;** The&nbsp;script&nbsp;does&nbsp;assume&nbsp;that&nbsp;the&nbsp;job&nbsp;has&nbsp;already&nbsp;been&nbsp;created&nbsp;by&nbsp;another&nbsp;means&nbsp;-&nbsp;in&nbsp;this&nbsp;case,&nbsp;by&nbsp;the [MaintenanceSolution.sql](https://github.com/olahallengren/sql-server-maintenance-solution/blob/master/MaintenanceSolution.sql) script, so you're wanting to configure your own SQL Agent jobs using this script, make sure they already exist.


### Script Deployment

Add the contents of[&nbsp;SCCM SQL Ola Hallengren Maintenance Scripts.ps1](https://github.com/jrudlin/SCCM-SQL-Maintenance/blob/master/SCCM%20SQL%20Ola%20Hallengren%20Maintenance%20Scripts.ps1) to the Discovery script on a new Configuration Item in SCCM:

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/1.compliance.jpg" alt="" class="wp-image-322"></figure>

Add a dummy compliance rule:

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/2.compliance.jpg" alt="" class="wp-image-323"></figure>

Add the CI to a new Baseline and deploy the baseline to your SCCM SQL/WSUS servers collection. I used the below two queries for my collection:

```TSQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client from SMS_R_System where SMS_R_System.SystemRoles = "SMS SQL Server"
```

```TSQL
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client from SMS_R_System where SMS_R_System.SystemRoles = "SMS Software Update Point"
```

## Results

Not much to look at, but from the SSMS we can see the agent jobs and the schedules have been configured in accordance with the settings in our CI script:

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/sqlagentjobs.jpg" alt="" class="wp-image-325"></figure>

- <figure><img src="{{ site.baseurl }}/assets/images/jobschedules.jpg" data-id="324" class="wp-image-324"></figure>

Your SQL databases both SYSTEM and USER databases will now be optimised using Ola's maintenance scripts.

Monitor the jobs in your test environment prior to production deployment to evaluate performance impact and competition times. The IndexOptimise job will take significantly longer during the first run if no maintenance has been performed on the databases before.