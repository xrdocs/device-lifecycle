---
published: true
date: '2016-10-14 12:38 -0700'
title: IOS-XR Packages and Security
author: Patrick Warichet
excerpt: securing RPM installation on IOS-XR
tags:
  - iosxr
  - cisco
  - linux
  - RPM
---

{% include toc icon="table" title="IOS-XR Packages and Security" %}

## Introduction

With the Introduction of IOX XR 6.0, the complete IOS XR software architecture has migrated to a open source infrastructure centered around the Linux Operating System. With the Adoption of Linux and Linux Containers (LXCs) some major change have been introduce in several areas; The monolithic set of XR features has been dis-aggregated in a collection of RPM packages that can be upgraded trough local or remote repositories. The Linux environment comes with its own authentication mechanism and user privileges. With XR and Linux processes residing side-by-side inside the same namespace, it is important to control the installation and execution of these processes.

### Packages Verification

Packages and ISOs are delivered in a tar file. included in that tar file is a readme file that contain a MD5 sum for all the files. MD5 sum are a good way to verify the integrity of the file using the md5sum utility on Unix.  md5sum generate a 128-bit value known as the "digital fingerprint." If two files have different MD5 sums, the files are definitely different whereas if two files have the same MD5 sum, it is highly likely the files are exactly alike. MD5 is technically a cryptography hash function which essentially means that the risk of producing collisions is really low but possible.

Example of verifying md5sum from the Linux shell using unnamed pipes.

```shell
cisco@galaxy-42:~$ diff -awsy <(grep mgbl README-ncs5k-k9sec-6.0.0 | cut -d' ' -f1) <(md5sum ncs5k-mgbl-2.0.0.0-r600.x86_64.rpm-6.0.0 | cut -d' ' -f1)
7a0e22ea86622dfc2293179d6fb52721               7a0e22ea86622dfc2293179d6fb52721
Files /dev/fd/63 and /dev/fd/62 are identical
```

or using a simple script:

```bash
!/bin/bash
if [ -z $3 ]; then
	echo "file + readme + feature needed"
	echo "usage: 0? "
	exit
fi
export csum=$(grep $3 $2 | cut -d' ' -f1)
export fsum=$(md5sum $1 | cut -d' ' -f1)
echo $csum
echo $fsum

if [ "$csum" = "$fsum" ]; then
	echo "MD5sum are identical"
else
	echo "MD5sum are different"
fi
```

```shell
cisco@galaxy-42:~$ chkmd5.sh ncs5k-mgbl-2.0.0.0-r600.x86_64.rpm-6.0.0 README-ncs5k-k9sec-6.0.0 mgbl
7a0e22ea86622dfc2293179d6fb52721
7a0e22ea86622dfc2293179d6fb52721
MD5sum are identical
```

### Package Signature

IOS XR 6.0.0 and above use the RPM format for all its packages, the RPM format include the capability to digitally sign packages using a SHA-1 key. At this time none of the IOS XR packages are signed, this feature will be available in a future release.

Some third party packages embed a RSA/SHA-1signature, to install these packages, the public key of the provider can be verified before and during package installation.

#### Example for the puppet client

**1) Key Presence**

On the local repository server, you can verify if the package has been signed by the provider.

```shell
cisco@galaxy-42:~$ rpm -qpi  puppet-agent-1.4.1-1.cisco_wrlinux7.x86_64.rpm | grep Signature
warning: puppet-agent-1.4.1-1.cisco_wrlinux7.x86_64.rpm: Header V4 RSA/SHA1 Signature, key ID 4bd6ec30: NOKEY
Signature   : RSA/SHA1, Thu 24 Mar 2016 03:05:33 PM PDT, Key ID 1054b7a24bd6ec30
```

**2) Key Verification**

On the local repository download and import the public key from the package provider, this step can be performed multiple time to verify the integrity of all packages in the local repository.

```shell
cisco@galaxy-42:~$ wget http://yum.puppetlabs.com/RPM-GPG-KEY-puppetlabs
cisco@galaxy-42:~$ rpm --import RPM-GPG-KEY-puppetlab
cisco@galaxy-42:~$ rpm -Kv  puppet-agent-1.4.1-1.cisco_wrlinux7.x86_64.rpm
puppet-agent-1.4.1-1.cisco_wrlinux7.x86_64.rpm:
     Header V4 RSA/SHA1 Signature, key ID 4bd6ec30: OK
     Header SHA1 digest: OK (fd954ea78e24ea32fdbaf3be045087ffe4c277ae)
     V4 RSA/SHA1 Signature, key ID 4bd6ec30: OK
     MD5 digest: OK (5eb0292058ba82449b7fb6eaa62fc102)
```

**3) Create a Repository Pointer**

On the Router, create a .repo file in /etc/yum/repos.d that enable key verification of packages, keys can also be copied on a local repository if this repository is secure.

```
[puppetlabs]
name=puppetlabs
baseurl=http://galaxy-42.cisco.com/Packages
gpgcheck=1
gpgkey=http://yum.puppetlabs.com/RPM-GPG-KEY-puppetlabs
enabled=1
```

**4) Package Installation from local repository**

```
xr-vm_node0_RP0_CPU0:~# yum install puppet-agent

Loaded plugins: app_plugin, downloadonly, protect-packages
    puppetlabs                                                                     | 2.9 kB     00:00
    Setting up Install Process
    Resolving Dependencies
    --> Running transaction check
    ---> Package puppet-agent.x86_64 0:1.4.1-1.cisco_wrlinux7 will be installed
    --> Finished Dependency Resolution
    Dependencies Resolved
    ======================================================================================================
    Package                Arch             Version                           Repository            Size
    ======================================================================================================
    Installing:
    puppet-agent           x86_64           1.4.1-1.cisco_wrlinux7            puppetlabs            41 M
    Transaction Summary
    ======================================================================================================
    Install       1 Package
    Total download size: 41 M
    Installed size: 143 M
    Is this ok [y/N]: y
    Retrieving key from http://yum.puppetlabs.com//RPM-GPG-KEY-puppetlabs
    Importing GPG key 0x4BD6EC30:
    Userid: "Puppet Labs Release Key (Puppet Labs Release Key) <info@puppetlabs.com>"
    From  : http://yum.puppetlabs.com//RPM-GPG-KEY-puppetlabs
    Is this ok [y/N]: y
    Downloading Packages:
    puppet-agent-1.4.1-1.cisco_wrlinux7.x86_64.rpm                                 |  41 MB     00:01
    Running Transaction Check
    Running Transaction Test
    Transaction Test Succeeded
    Running Transaction
      Installing : puppet-agent-1.4.1-1.cisco_wrlinux7.x86_64                                         1/1
    Installed:
      puppet-agent.x86_64 0:1.4.1-1.cisco_wrlinux7
    Complete!
```
