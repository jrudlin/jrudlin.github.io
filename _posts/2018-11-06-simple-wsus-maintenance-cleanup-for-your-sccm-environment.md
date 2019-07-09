---
layout: post
title: Simple WSUS Maintenance/Cleanup for your SCCM environment
date: 2018-11-06 13:27:27.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Automation
- PowerShell
- SCCM
- WSUS
tags:
- Configuration baseline
- Configuration Item
- SCCM software updates
- WSUS Cleanup
- WSUS Maintenance
- WSUS PowerShell cleanup
meta:
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1059799451513044995";}}
  timeline_notification: '1541510851'
  _rest_api_published: '1'
  _rest_api_client_id: "-1"
  _publicize_job_id: '23977726462'
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
permalink: "/2018/11/06/simple-wsus-maintenance-cleanup-for-your-sccm-environment/"
---

WSUS Cleanup in an SCCM has been done before - many times but I wanted a solution that focussed purely on the built-in WSUS cleanup options and something that was **simple to deploy** in any SCCM environment - using an **SCCM Configuration Item**.

# WSUS Cleanup - SCCM built-in maintenance

Windows Server Updates Services (WSUS) is the underlying service used by System Center Configuration Manager (SCCM) to perform software updates (Windows patches for software and OS) on Windows servers and workstations.

The database that WSUS uses to store metadata about all the available updates has become very large over the years, covering more and more Microsoft operating systems and software. It is highly recommended by Microsoft to **perform regular WSUS database maintenance** as suggested here: [Complete guide to Microsoft WSUS and Configuration Manager SUP maintenance](https://blogs.technet.microsoft.com/configurationmgr/2016/01/26/the-complete-guide-to-microsoft-wsus-and-configuration-manager-sup-maintenance/).

When WSUS is used in conjunction with SCCM, SCCM can perform some automated cleanup tasks after each WSUS sync job has completed. In SCCM 1806 this cleanup is further improved and documented here: [Software Updates Maintenance](https://docs.microsoft.com/en-us/sccm/sum/deploy-use/software-updates-maintenance). Specifically:

> _Starting in version 1806, the **WSUS cleanup option occurs after every sync** and does the following cleanup items:_

> _The **Expired updates** option for WSUS servers on CAS and primary sites._
> 
> - _WSUS servers for secondary sites, don't run WSUS cleanup for expired updates._  
> 
> _Configuration Manager builds a list of superseded updates from its database. The list is based on the supersedence behavior in the Software Update Point component properties._
> 
> - _The update configuration items meeting the supersedence behavior criteria are expired in the Configuration Manager console._
> - _The updates are declined in WSUS for CAS and primary sites but not for secondary sites._
> 
> _A cleanup for software update configuration items in the Configuration Manager database occurs every seven days and removes unneeded updates from the console._
> 
> - This cleanup won't remove expired updates from the Configuration Manager console if they're currently deployed.

The document then goes on to state:

> _The following **WSUS Server Cleanup Wizard** options_ **aren't run** _on the CAS and primary sites:_
> 
> - _Unused updates and update revisions_
> - _Computers not contacting the server_
> - _Unneeded update files_

The WSUS Cleanup option in SCCM is exposed here (in the configuration of the root SCCM sites' SUP properties) :  
 ![WSUS Cleanup]({{ site.baseurl }}/assets/images/wsus-cleanup.jpg)

Unfortunately what the Microsoft documentation does not cover is the order in which to run the cleanup jobs and on which level of the WSUS hierarchy.

In an SCCM hierarchy there are usually multiple tiers of WSUS:

1. A single root WSUS (The top tier/level) which updates from the internet.
2. Multiple lower level WSUS servers that sync with the root level WSUS.

Note: **WSUS servers can also share the same SQL database** so the maintenance would only need to be run once per database, rather than once per WSUS server, not that it would harm anything by running it multiple times per database.

There are comments from those in the know that certain WSUS cleanup tasks [recommended-order-for-running-cleanup-wizard-in-wsus-upstream-replica-hierarchy](https://social.technet.microsoft.com/Forums/lync/en-US/15c83bff-610e-4120-bfa8-979aabf3f78c/recommended-order-for-running-cleanup-wizard-in-wsus-upstream-replica-hierarchy?forum=winserverwsus) should only be run from the top level WSUS down, so to clarify for sure, I went ahead to see **how the SCCM team had implemented WSUS cleanup in a hierarchy** by analysing the wsyncmgr.log:

1. The CAS server (\*VM01\*) initiates a sync on the top level WSUS - (\*565):  
 ![WSUS_01]({{ site.baseurl }}/assets/images/wsus_011.png)
2. Once the WSUS sync with the Microsoft Update catalog is done, SCCM syncÔÇÖs itÔÇÖs database with WSUS:  
 ![WSUS_02]({{ site.baseurl }}/assets/images/wsus_02.png)
3. Cleanup is then performed on the top tier WSUS. Cleanup in this case is marking updates as ÔÇÿdeclinedÔÇÖ ÔÇô this is important:  
 ![WSUS_03]({{ site.baseurl }}/assets/images/wsus_03.png)
4. At this stage, nothing has happened with the lower tier WSUS servers. The CAS now updates the Primary Sites inboxes, requesting a WSUS Sync to the top tier WSUS:  
 ![WSUS_04]({{ site.baseurl }}/assets/images/wsus_041.png)
5. So far all the above snippets are from the CAS. Now we jump to a Primary Site server. We can see the Sync has started, 1 minute after the sync and cleanup has finished on the CAS.  
 ![WSUS_05]({{ site.baseurl }}/assets/images/wsus_05.png)
6. The Sync on the Primary site completes and run its own cleanup:  
 ![WSUS_06]({{ site.baseurl }}/assets/images/wsus_06.png)

So above we can see that SCCM is processing the ' **decline** updates' part of the **cleanup in the correct order - from the TOP WSUS, down**.

Now to process the remaining WSUS cleanup options that Microsoft have stated **SCCM doesn't do** :

> - _Unused updates and update revisions_
> - _Computers not contacting the server_
> - _Unneeded update files_

## The Custom Cleanup script

I have taken some Powershell from [@Bdam55](https://twitter.com/bdam555)┬áto cover the above three WSUS cleanup tasks and converted it to run as an SCCM Configuration Item on a schedule:

[SCCM WSUS Cleanup script on Github](https://github.com/jrudlin/SCCM-WSUS_Cleanup)

The actual cleanup of WSUS is nothing new, the PowerShell used here is regularly available on the internet, the cool thing is that this is a CI and is running in the SYSTEM context.

The script is designed to run against WSUS in an SCCM 1806 and above environment, as the additional WSUS Cleanup rules were introduced in this version.

Here are the **benefits** of this deployment method of the cleanup script, over say something like Group Policy Scheduled Tasks or BigFix scheduled script:

- A Configuration Baseline can be targeted at a dynamic device collection containing all the SCCM site servers in the hierarchy.
- All SCCM infrastructure maintenance is kept and controlled within SCCM.
- No credentials are required in the script as it runs as SYSTEM on the SCCM Site Server
- Uses the ConfigMgr PoSh module on the Site Server and then passes variables to the remote WSUS

When the script runs on the SCCM Site Server, it actually connects to each of the SUP (WSUS) servers in the site using PowerShell remoting (invoke-command) and initiates a cleanup if a sync job is no running. If the script detects it's running on the root WSUS server, it will add a large delay in processing the cleanup, to give the lower level WSUS cleanup jobs time to finish.

Top level WSUS detection:

[code language="powershell"]  
$TopTierWsus = Get-CMSoftwareUpdateSyncStatus | Where-Object -FilterScript {$\_.WSUSSourceServer -like "\*Microsoft Update\*" -and $\_.SiteCode -eq $SiteCode} | Select-Object -Unique -ExpandProperty WSUSServerName&amp;lt;span data-mce-type="bookmark" id="mce\_SELREST\_start" data-mce-style="overflow:hidden;line-height:0" style="overflow:hidden;line-height:0" &amp;gt;&amp;lt;/span&amp;gt;  
[/code]

As per [https://blogs.technet.microsoft.com/meamcs/2018/10/09/resolving-wsus-performance-issues/](https://blogs.technet.microsoft.com/meamcs/2018/10/09/resolving-wsus-performance-issues/) the script now also configures the minimum values for:

- WSUS App Pool queue length
- WSUS App Pool memory size

[code language="powershell"]  
# WSUS IIS AppPool Variables  
$WSUSSiteNameFilter = "WSUS\*"  
$IISAppPoolQueueMinSize = 2000  
$IISAppPoolMemoryMinSize = 4194304  
$ISSWebAdminModuleNAme = "WebAdministration"

$IISModule = $true  
If(-not(get-module -Name $ISSWebAdminModuleNAme)){  
 Try{  
 Import-Module -Name $ISSWebAdminModuleNAme  
 } Catch {  
 $IISModule = $false  
 }  
}

If($IISModule){

$WebSite = Get-Website | Where-Object -FilterScript {$\_.Name -like $WSUSSiteNameFilter}  
 If(-not($WebSite)){Add-TextToCMLog -LogFile $LogFile -Value "Could not find IIS website using filter '$WSUSSiteNameFilter'. Skipping IIS config" -Component $component -Severity 3}

$WSUSAppPoolName = $WebSite.applicationPool

$AppPool = Get-ItemProperty -Path IIS:\AppPools\$WSUSAppPoolName

# WSUS App Pool Queue Length  
 $AppPoolQueueLength = $AppPool.queueLength

If($AppPoolQueueLength -lt $IISAppPoolQueueMinSize){  
 Add-TextToCMLog -LogFile $LogFile -Value "Setting $WSUSAppPoolName IIS App Pool length to $IISAppPoolQueueMinSize" -Component $component -Severity 1  
 Set-ItemProperty -Path $AppPool.PSPath -Name queueLength -Value $IISAppPoolQueueMinSize  
 } else {  
 Add-TextToCMLog -LogFile $LogFile -Value "$WSUSAppPoolName IIS App Pool length is $AppPoolQueueLength so is already above $IISAppPoolQueueMinSize" -Component $component -Severity 1  
 }

# WSUS App Pool Private Memory Size  
 $AppPoolMemorySize = (Get-ItemProperty -Path $AppPool.PSPath -Name recycling.periodicrestart.privateMemory).Value

If($AppPoolMemorySize -lt $IISAppPoolMemoryMinSize){  
 Add-TextToCMLog -LogFile $LogFile -Value "Setting $WSUSAppPoolName IIS App Pool memory size to $IISAppPoolMemoryMinSize" -Component $component -Severity 1  
 Set-ItemProperty -Path $AppPool.PSPath -Name recycling.periodicrestart.privateMemory -Value $IISAppPoolMemoryMinSize  
 } else {  
 Add-TextToCMLog -LogFile $LogFile -Value "$WSUSAppPoolName IIS App Pool memory is $AppPoolMemorySize so is already above $IISAppPoolMemoryMinSize" -Component $component -Severity 1  
 }

}  
else  
{  
 Add-TextToCMLog -LogFile $LogFile -Value "Could not load module $ISSWebAdminModuleNAme. Skipping IIS config" -Component $component -Severity 3  
}  
&amp;lt;span id="mce\_SELREST\_start" style="overflow: hidden; line-height: 0;"&amp;gt;&amp;lt;/span&amp;gt;[/code]

If the SCCM hierarchy is expanded to cope with additional load, or new servers are brought online to do a side-by-side migration, the Configuration Item should kick-in and cover these off automatically.

Thanks to: [@Bdam55](https://twitter.com/bdam555) for the [Add-TextToCMLog](https://damgoodadmin.com/2018/04/10/so-you-want-to-write-a-powershell-script/) function and most of the [WSUS cleanup script](https://damgoodadmin.com/2017/11/05/fully-automate-software-update-maintenance-in-cm/).

Any questions, feel free to reach out.

Thanks

