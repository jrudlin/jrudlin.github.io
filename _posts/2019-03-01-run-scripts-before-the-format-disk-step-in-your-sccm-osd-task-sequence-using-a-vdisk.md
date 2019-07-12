---
layout: post
title: Run scripts before the 'Format Disk' step in your SCCM OSD Task Sequence using
  a vdisk
date: 2019-03-01 17:19:55.000000000 +00:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- OSD
- SCCM
- Windows 10
tags:
- ConfigMgr
- format disk
- package content download
- Run scripts
- Task Sequence
meta:
  _wpas_skip_21561694: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:18662058;s:57:"https://twitter.com/JackRudlin/status/1101532538428973057";}}
  publicize_linkedin_url: ''
  _publicize_job_id: '28167815927'
  _thumbnail_id: '340'
  timeline_notification: '1551460796'
  _publicize_done_18837840: '1'
  _wpas_done_18662058: '1'
  publicize_twitter_user: JackRudlin
  _publicize_done_21557384: '1'
  _wpas_done_22181804: '1'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2019/03/01/run-scripts-before-the-format-disk-step-in-your-sccm-osd-task-sequence-using-a-vdisk/"
---
## Scenario

You've got your SCCM OSD Task Sequence ready/working but you want to run some scripts in the WinPE phase before the Format Disk steps.

This will generally work well if there is a formatted disk available as you can just reference your script in a package like you normally would, and the package content will be downloaded to the disk.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

But what if the disk is not formatted or BitLocker encrypted or is not writable for some other reason?

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Simple, just run them from a network share right? Like most people do. How about if you have 100's or even 1000's or remote locations with poor WAN connectivity and no local file servers/NAS etc.?

Well, look no further, the solution is herein detailed.

<!-- /wp:paragraph -->

<!-- wp:heading -->

## Existing Solution

<!-- /wp:heading -->

<!-- wp:paragraph -->

One of the most commonly used solutions out there right now is to run scripts directly from a UNC path (make sure you authenticate first):

<!-- /wp:paragraph -->

<!-- wp:image {"id":348} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/sccm-run-scripts-from-unc.jpg" alt="" class="wp-image-348"><br>
<figcaption>Running scripts from the network without package content download</figcaption>
</figure>

<!-- /wp:image -->

<!-- wp:heading -->

## New vdisk Solution

<!-- /wp:heading -->

<!-- wp:paragraph -->

The new solution creates a temporary vhdx file in the RAM drive that WinPE is using. This is an NTFS formatted virtual disk and can be used to download package content when there is no physical disk available because it first needs formatting.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

You may ask, why not just format the disk first and run the script? Well if you run some pre-checks and find the machine is not suitable/supported for building and want to revert back to the currently installed OS, then formatting it would not be a great thing to do.

<!-- /wp:paragraph -->

<!-- wp:image {"id":340} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/all-steps.jpg" alt="" class="wp-image-340"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### Temp Local Disk group

<!-- /wp:heading -->

<!-- wp:paragraph -->

Create a condition on the group so that it only creates the vdisk if no local disk is available - this condition is optional, there is no harm in creating the vdisk always, as it gets cleaned up shortly after anyway.

<!-- /wp:paragraph -->

<!-- wp:image {"id":345} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/temp-local-disk-group.jpg" alt="" class="wp-image-345"></figure>

<!-- /wp:image -->

<!-- wp:image {"id":344} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/temp-local-disk-group2.jpg" alt="" class="wp-image-344"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### Run Command Line - Create Temp vDisk DiskPart File

<!-- /wp:heading -->

<!-- wp:paragraph -->

In the WinPE phase, right at the beginning of the Task Sequence create a Run Command Line step with the following code (adjust the vdisk size according to the amount of content you need to download before the format disk step using **maximum=xxx** ).

**Note:&nbsp;** The&nbsp;space available in WinPE RAM drive is adjustable in the boot image properties.

<!-- /wp:paragraph -->

<!-- wp:preformatted -->

```
cmd.exe /c echo create vdisk file="%temp%\temp.vhdx" maximum=100 \>\> %temp%\diskpartvDisk.txt && cmd.exe /c echo select vdisk file="%temp%\temp.vhdx" \>\> %temp%\diskpartvDisk.txt && cmd.exe /c echo attach vdisk \>\> %temp%\diskpartvDisk.txt && cmd.exe /c echo create part primary \>\> %temp%\diskpartvDisk.txt && cmd.exe /c echo format fs=ntfs label="Temp Vol" quick \>\> %temp%\diskpartvDisk.txt && cmd.exe /c echo assign \>\> %temp%\diskpartvDisk.txt
```

<!-- /wp:preformatted -->

<!-- wp:image {"id":347} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/diskpart-cmds.jpg" alt="" class="wp-image-347"></figure>

<!-- /wp:image -->

<!-- wp:image {"id":346} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/diskpart-cmds2.jpg" alt="" class="wp-image-346"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### Run Command Line - Run DiskPart vDisk File

<!-- /wp:heading -->

<!-- wp:paragraph -->

Run diskpart.exe against the file created in the previous step

<!-- /wp:paragraph -->

<!-- wp:preformatted -->

```
diskpart.exe /s %temp%\diskpartvDisk.txt
```

<!-- /wp:preformatted -->

<!-- wp:image {"id":343} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/run-diskpart.jpg" alt="" class="wp-image-343"></figure>

<!-- /wp:image -->

<!-- wp:image {"id":342} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/run-diskpart2.jpg" alt="" class="wp-image-342"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### Use package content

<!-- /wp:heading -->

<!-- wp:paragraph -->

Run any script/file, whatever you want, using package content

<!-- /wp:paragraph -->

<!-- wp:image {"id":341} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/ui-prompt-package.jpg" alt="" class="wp-image-341"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### vDisk Cleanup Group

<!-- /wp:heading -->

<!-- wp:paragraph -->

Has the following condition

<!-- /wp:paragraph -->

<!-- wp:image {"id":339} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/cleanup-group.jpg" alt="" class="wp-image-339"></figure>

<!-- /wp:image -->

<!-- wp:image {"id":338} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/cleanup-group2.jpg" alt="" class="wp-image-338"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### Run Command Line - Create DiskPart File to del vDisk

<!-- /wp:heading -->

<!-- wp:paragraph -->

Run the following script to create a diskpart file that will cleanup the existing vdisk

<!-- /wp:paragraph -->

<!-- wp:preformatted -->

```
cmd.exe /c echo list volume\> %temp%\diskpartDelvDiskList.txt & for /f "tokens=1-4" %a in ('diskpart /s %temp%\diskpartDelvDiskList.txt ^| find "Temp Vol"') do echo select %a %b\> %temp%\diskpartDelvDisk.txt & echo clean\>\> %temp%\diskpartDelvDisk.txt & echo offline disk\>\> %temp%\diskpartDelvDisk.txt
```

<!-- /wp:preformatted -->

<!-- wp:image {"id":337} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/cleanup-diskpart-cmds.jpg" alt="" class="wp-image-337"></figure>

<!-- /wp:image -->

<!-- wp:image {"id":336} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/cleanup-diskpart-cmds2.jpg" alt="" class="wp-image-336"></figure>

<!-- /wp:image -->

<!-- wp:heading {"level":3} -->

### Run Command Line - Run DiskPart Del vDisk File

<!-- /wp:heading -->

<!-- wp:preformatted -->

```
diskpart.exe /s %temp%\diskpartDelvDisk.txt
```

<!-- /wp:preformatted -->

<!-- wp:image {"id":350} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/cleanup-diskpart.jpg" alt="" class="wp-image-350"></figure>

<!-- /wp:image -->

<!-- wp:image {"id":349} -->

<figure class="wp-block-image"><img src="{{ site.baseurl }}/assets/images/cleanup-diskpart2.jpg" alt="" class="wp-image-349"></figure>

<!-- /wp:image -->

<!-- wp:paragraph -->

So that's it, the OSD Task Sequence can continue with the format disk steps following the cleanup of the vdisk.

<!-- /wp:paragraph -->

