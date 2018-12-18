---
published: true
date: '2016-08-06 17:06 -0700'
title: IOS-XR and RPM Package Manager
author: Patrick Warichet
excerpt: Introduction to RPM Package Manager
tags:
  - iosxr
  - linux
  - rpm
  - yum
---
{% include toc icon="table" title="IOS-XR and RPM package manager" %}

## Introduction

With IOS XR 6.0, the Package Installation Envelope (PIE) format has been discarded in favor of the RPM Package Manager (RPM) format. This move aligns IOS XR 6.0  and above more closely with RPM-based Linux distribution like Red Hat or Centos. RPM is a free software project and is released under GPL. Briefly, RPMs contain the following elements:

1. CPIO archive: Contains the absolute path of all the package’s files;
2. Metadata: Contains the package dependencies;
3. Scriptlets: Scripts that perform pre and post (un)installation tasks;

The Metadata is a XML file that helps determine and resolve package dependencies, to facilitate dependencies resolution the metadata is used to poulate a small database located in /var/lib/rpm that can be queried using tools like YUM or RPM.
A Software Maintenance Update (SMU) will be published in the form of tape archive (tar) format. The SMU will contain the following files:

1. A Readme.txt file describing the content of the SMU;
2. One or more RPMs;
3. A Package-mdata.txt that contains a MD5 checksum of all the packages in the tar file;

![Anatomy of RPM]({{site.baseurl}}/images/RPM.png)

## XR Packages Installation 
Within IOS-XR, two new CLI commands have been introduced that complement the existing ones: "install update" and "install upgrade", described Table 1. These new commands require an external packages repository accessible through FTP/SFTP/SCP/TFTP or HTTP.

| Command   | Description |
| install update source <repository> | When no package is specified, update latest SMUs of all installed packages. |
| install upgrade source <repository> version <ver_num> | Upgrade the base image to the specified version. All installed packages are upgraded to same release as the base package. |

```
RP/0/RP0/CPU0:pwa-rtr#install update source ?
   WORD  Enter source directory for the package(s)   
         Example: 
          sftp://user@server/directory/
          scp://user@server/directory/
          ftp://user@server/directory/
          tftp://server/directory/
          http://server/directory/
```
In the example below the k9sec package is installed using the "install update" command. After initiating the command, you can issue a "show install request" to monitor the status of the package installation.

```
RP/0/RP0/CPU0:pwa-rtr#install update source http://192.168.122.1:8080/xrv9k xrv9k-k9sec
Sat Feb 13 17:18:59.981 UTC
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Update in progress...
Scheme : http
Hostname : 192.168.122.1:8080
Collecting software state..
Update packages :
    xrv9k-k9sec
Fetching .... xrv9k-k9sec-1.0.0.0-r600.x86_64.rpm-6.0.0
Adding packages
    xrv9k-k9sec-1.0.0.0-r600.x86_64.rpm-6.0.0
Feb 13 17:19:08 Install operation 22 started by root:
install add source /misc/disk1/install_tmp_staging_area/6.0.0 xrv9k-k9sec-1.0.0.0-r600.x86_64.rpm-6.0.0
Feb 13 17:19:09 Install operation will continue in the background

Feb 13 17:19:12 Install operation 20 finished successfully
Install add operation successful
Activating xrv9k-k9sec-1.0.0.0-r600
Feb 13 17:19:14 Install operation 23 started by root:
  install activate pkg xrv9k-k9sec-1.0.0.0-r600
Feb 13 17:19:14 Package list:
Feb 13 17:19:14     xrv9k-k9sec-1.0.0.0-r600
Feb 13 17:19:16 Install operation will continue in the background
 
RP/0/RP0/CPU0:pwa-rtr#
 
 
This product contains cryptographic features and is subject to United
States and local country laws governing import, export, transfer and
use. Delivery of Cisco cryptographic products does not imply third-party
authority to import, export, distribute or use encryption. Importers,
exporters, distributors and users are responsible for compliance with
U.S. and local country laws. By using this product you agree to comply
with applicable laws and regulations. If you are unable to comply with
U.S. and local laws, return this product immediately.

A summary of U.S. laws governing Cisco cryptographic products may be
found at:
http://www.cisco.com/wwl/export/crypto/tool/stqrg.html

If you require further assistance please contact us by sending email to
export@cisco.com.

Feb 13 17:20:12 Install operation 21 finished successfully
```

All the install commands log their progress. You can enter "show install log" to review the installation log file. This command is useful for identifying the reason for any failure.

```
RP/0/RP0/CPU0:pwa-rtr#show install log 21
Tue Feb 23 17:54:00.961 UTC
Feb 23 17:51:14 Install operation 21 started by root:
      install activate pkg xrv9k-k9sec-1.0.0.0-r600
Feb 23 17:51:14 Package list:
Feb 23 17:51:14     xrv9k-k9sec-1.0.0.0-r600
Feb 23 17:51:15 Action 1: install prepare action started
Feb 23 17:51:17 Install operation will continue in the background
Feb 23 17:51:17 The prepared software is set to be activated with process restart
Feb 23 17:51:18 Start preparing software for local installation
Feb 23 17:51:19 Action 1: install prepare action finished successfully
Feb 23 17:51:20 Action 2: install activate action started
Feb 23 17:51:20 The software will be activated with process restart
Feb 23 17:51:22 Activating XR packages
Feb 23 17:51:58 0 processes affected at node 0/RP0/CPU0
Feb 23 17:52:10 Action 2: install activate action finished successfully
Feb 23 17:52:11 Install operation 21 finished successfully
Feb 23 17:52:11 Ending operation 21
Install add operation successful
```

## XR Package Structure 
When you enter the Shell of the control plane (accessed by typing "bash" or "run" from the CLI) you are in a full Bash shell environment of the XR container with all the traditional Linux tools at your fingertips. (You can return to the IOS XR CLI by typing “exit”).
Since all IOS XR packages used the RPM format, we will use the Linux rpm utility to inspect their content. The “-q” switch used below is used to query the installed packages database and the “-i” switch display the general information contained in the metadata of the RPM package.

```
RP/0/RP0/CPU0:pwa-rtr#run
Wed Dec  2 03:04:32.231 UTC
 [xr-vm_node0_RP0_CPU0:~]$rpm -qi xrv9k-k9sec
Name        : xrv9k-k9sec        Relocations: /opt/cisco/XR/packages/xrv9k-k9sec-1.0.0.0-r600
Version     : 1.0.0.0                           Vendor: (none)
Release     : r600                          Build Date: Thu Dec 24 08:46:14 2015
Install Date: Fri Mar  4 12:18:15 2016      Build Host: iox-lnx-009
Group       : IOS-XR                        Source RPM: xrv9k-k9sec-1.0.0.0-r600.src.rpm
Size        : 8616918                License: Copyright (c) 2015 Cisco Systems Inc. All rights reserved.
Signature   : (none)
Packager    : alnguyen
Summary     : Bundle package for iosxr-security
Architecture: x86_64
Description :
Bundle package for iosxr-security
Build workspace: /auto/srcarchive16/production/6.0.0/xrv9k/workspace
```

We can also use RPM utilities to query the requirement of the package (“-R” switch), this information is also in the RPM metadata and is crucial for dependency checking. In the example below, the k9sec package depends on three packages, each of them within a certain version range.

```
[xr-vm_node0_RP0_CPU0:~]$rpm -qR xrv9k-k9sec
/bin/sh
/bin/sh
/bin/sh
/bin/sh
xrv9k-iosxr-fwding >= 1.0.0.0
xrv9k-iosxr-fwding < 2.0.0.0
xrv9k-iosxr-infra >= 1.0.0.0
xrv9k-iosxr-infra < 2.0.0.0
xrv9k-iosxr-os >= 1.0.0.0
xrv9k-iosxr-os < 2.0.0.0
```

All the Cisco packages are in a separate group named IOS-XR. With the “—queryformat” switch we specify the type of field we want to display and how we want to display them. The following command will query all the installed packages and display their groups and names we use the utility grep to filter the output and only display the package belonging to the IOS XR group. Enter the following command to query all the packages from that group:

```
[xr-vm_node0_RP0_CPU0:~]$rpm -qa --queryformat '%{group}   %{name}\n' | grep IOS-XR
IOS-XR   xrv9k-spirit-boot
IOS-XR   xrv9k-iosxr-routing
IOS-XR   xrv9k-iosxr-fwding
IOS-XR   xrv9k-iosxr-infra
IOS-XR   xrv9k-base
IOS-XR   xrv9k-bgp
IOS-XR   xrv9k-common-pd-fib 
IOS-XR   xrv9k-fwding
IOS-XR   xrv9k-gcp-fwding
IOS-XR   xrv9k-gdplane
IOS-XR   xrv9k-iosxr-os
IOS-XR   xrv9k-os-support
IOS-XR   xrv9k-k9sec
IOS-XR   xrv9k-mgbl
```

With the Help of standard Linux tools we can review the full history of installed IOS-XR packages:

```
[xr-vm_node0_RP0_CPU0:~]$rpm -qa --queryformat '%{installtime} %{group} %{name}-%{version}-%{release} %{installtime:date}\n' | grep IOS-XR | sort -nr | sed -e 's/^[0-9]*\ IOS-XR\ //'
xrv9k-mgbl-2.0.0.0-r600 Fri Mar  4 12:33:59 2016
xrv9k-k9sec-1.0.0.0-r600 Fri Mar  4 12:18:15 2016
xrv9k-os-support-1.0.0.0-r600 Fri Jan 29 02:04:37 2016
xrv9k-iosxr-os-1.0.0.0-r600 Fri Jan 29 02:04:35 2016
xrv9k-gdplane-1.0.0.0-r600 Fri Jan 29 02:04:33 2016
xrv9k-gcp-fwding-1.0.0.0-r600 Fri Jan 29 02:04:33 2016
xrv9k-fwding-1.0.0.0-r600 Fri Jan 29 02:04:32 2016
xrv9k-common-pd-fib-1.0.0.0-r600 Fri Jan 29 02:04:32 2016
xrv9k-bgp-1.0.0.0-r600 Fri Jan 29 02:04:31 2016
xrv9k-base-1.0.0.0-r600 Fri Jan 29 02:04:31 2016
xrv9k-iosxr-infra-1.0.0.0-r600 Fri Jan 29 02:04:27 2016
xrv9k-iosxr-fwding-2.0.0.0-r600 Fri Jan 29 02:04:25 2016
xrv9k-spirit-boot-1.0.0.0-r600 Fri Jan 29 02:04:24 2016
xrv9k-iosxr-routing-1.0.0.0-r600 Fri Jan 29 02:04:24 2016
```

With the “—requires” switch we can query the RPM database and display the version of installed packages that are fulfilling the dependency of another package. Using the following command we learn which packages requires xrv9k-iosxr-routing to be present in the system:

```
[xr-vm_node0_RP0_CPU0:~]$rpm -q --whatrequires xrv9k-iosxr-routing
xrv9k-iosxr-fwding-2.0.0.0-r600.x86_64
xrv9k-bgp-1.0.0.0-r600.x86_64
xrv9k-mgbl-2.0.0.0-r600.x86_64
```

With following command we get the version of the package that provides the xrv9k-iosxr-routing functionality:

```
[xr-vm_node0_RP0_CPU0:~]$rpm -q --whatprovides xrv9k-iosxr-routing
  xrv9k-iosxr-routing-1.0.0.0-r600.x86_64
```

## Packages Installation from the Shell

You can use the install commands inside the Linux shell of the IOS-XR container to install packages using a shell script.

```
[xr-vm_node0_RP0_CPU0:~]$#install update source http://192.168.122.1:8080/xrv9k xrv9k-eigrp
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Update in progress...
Scheme : http
Hostname : 192.168.122.1:8080
Collecting software state..

Update packages :
	xrv9k-eigrp
Fetching .... xrv9k-eigrp-1.0.0.0-r600.x86_64.rpm-6.0.0
Adding packages 
	xrv9k-eigrp-1.0.0.0-r600.x86_64.rpm-6.0.0
Mar 10 22:40:38 Install operation 24 started by root:
 install add source /misc/disk1/install_tmp_staging_area/6.0.0 xrv9k-eigrp-1.0.0.0-r600.x86_64.rpm-6.0.0 
Mar 10 22:40:40 Install operation will continue in the background
Mar 10 22:40:43 Install operation 24 finished successfully

Install add operation successful
Activating xrv9k-eigrp-1.0.0.0-r600
Mar 10 22:40:45 Install operation 30 started by root:
  install activate pkg xrv9k-eigrp-1.0.0.0-r600 
Mar 10 22:40:45 Package list:
Mar 10 22:40:45     xrv9k-eigrp-1.0.0.0-r600
Mar 10 22:40:48 Install operation will continue in the background
Mar 10 22:40:43 Install operation 24 finished successfully
Install add operation successful
Activating xrv9k-eigrp-1.0.0.0-r600
Mar 10 22:40:45 Install operation 25 started by root:
  install activate pkg xrv9k-eigrp-1.0.0.0-r600 
Mar 10 22:40:45 Package list:
Mar 10 22:40:45     xrv9k-eigrp-1.0.0.0-r600
Mar 10 22:40:48 Install operation will continue in the background
Mar 10 22:41:49 Install operation 25 finished successfully
```

## Analyzing Package Using Linux

Any Linux distribution installed with the RPM utilities allows you to look at the content of packages, this is very useful to analyze dependencies and verify package integrity outside of the router. In this example we use the “-l” switch to display the full path of all the files inside the package.
NOTE: Notice the extra -p switch used to query uninstalled packages.

```shell
cisco@compute:~$ cd ~/web_server/xrv9k/
cisco@compute:~/web_server/xrv9k$ rpm -qpl xrv9k-k9sec-1.0.0.0-r600.x86_64.rpm-6.0.0
/
/opt
/opt/cisco
/opt/cisco/XR
/opt/cisco/XR/packages
/opt/cisco/XR/packages/xrv9k-k9sec-1.0.0.0-r600
/opt/cisco/XR/packages/xrv9k-k9sec-1.0.0.0-r600/all
/opt/cisco/XR/packages/xrv9k-k9sec-1.0.0.0-r600/all/etc
/opt/cisco/XR/packages/xrv9k-k9sec-1.0.0.0-r600/all/etc/compat-mdata
<SNIP>
cisco@compute:~/web_server/xrv9k$ rpm -qpR xrv9k-k9sec-1.0.0.0-r600.x86_64.rpm-6.0.0
/bin/sh
/bin/sh
/bin/sh
/bin/sh
xrv9k-iosxr-fwding >= 2.0.0.0
xrv9k-iosxr-fwding < 3.0.0.0
xrv9k-iosxr-infra >= 1.0.0.0
xrv9k-iosxr-infra < 2.0.0.0
xrv9k-iosxr-os >= 1.0.0.0
xrv9k-iosxr-os < 2.0.0.0
```

---
NOTE: Run these RPM utilities off-box on any Linux system that has the RPM utility installed. Experiment with some of the RPM commands to create your own dependency management tool for XR packages.
---
