---
published: true
date: '2016-10-20 14:47 -0700'
title: Using ZTP to install Puppet
tags:
  - iosxr
  - linux
  - puppet
  - ZTP
author: Patrick Warichet
excerpt: Installation of the puppet agent using ZTP
---

{% include toc icon="table" title="Using ZTP to install Puppet" %}

## Introduction
Puppet is a configuration management tool for information technology (IT) professionals. Puppet can potentically manage any resource defined on a node. Puppet can manage complex and distributed components to ensure service consistency and availability. In short, Puppet uses a configuration policy referred to as a "recipe" to bring systems into compliance.

Puppet enables you to make a lot of changes both quickly and consistently. Unlike scripts, you don’t have to write out every step, you only have to define how it should be. You are not required to write out the process for evaluating and verifying different conditions. Instead, you utilize the Puppet configuration language to declare the final state of the resources. This is why Puppet is often described as a declarative language.

The difference between a declarative language and a procedural language is: You tell Puppet the desired end results, but not the steps to get there.

If you choose to use Puppet as your configuration platform, it is important to have the agent installed on the devices early on in the deployment process, and not use any scripting beyond the installation procedure of the agent.

## Puppet in IOS-XR
Puppetlabs have created a puppet agent for the IOS-XR Yocto based Linux. You can follow their installation instructions here:
Installing Cisco IOS-XR agents "Installing Cisco IOS-XR agents"). Make sure you download the file which has “cisco-wrlinux-7” in the filename, since this is the version that works in the IOS-XR Yocto Linux environment.

If you are interested in using Puppet in conjunction with YANG data models, take a look at the tutorial on Puppet and ciscoyang [Using Puppet with IOS-XR 6.1.1](https://xrdocs.github.io/application-hosting/tutorials/2016-08-22-using-puppet-with-iosxr-6-1-1 "Using Puppet with IOS-XR 6.1.1"). You will have to modify the example below to include the installation of the GRPC GEM file.

## Script Example
Here is a small example on how to use ZTP to download and install the Puppet package from a local repository, configure the device hostname and enable DNS resolution. The Puppet agent package is secured with a SHA-1 (GPG) key, and you will need to download it to a secure location or access the key directly from Puppetlabs. In the example below, I simply placed the key in the same directory as the RPM package.
Puppet requires all hostnames to be resolved. The script relies on a DNS server to resolve the hostname and include the modifications to IOS-XR Linux. The script also assumes that communication between the Puppet master and the agent will happen over the management interface.

```
#!/bin/bash

YUM_REPO="http://172.30.0.22/packages/puppet"
YUM_PUPPET="/etc/yum/repos.d/puppet.repo"
PUPPET_SRV="puppet-master.cisco.local"
PUPPET_CONF="/etc/puppetlabs/puppet/puppet.conf"
DOMAIN="cisco.local"
DOMAIN_SRV=172.30.0.25
HOSTNAME="ncs-5001-c"
MGMT_IP="172.30.12.54 255.255.255.0"

source ztp_helper.sh

function create_repo(){
   # Create local repository file for puppet
   echo "### created by ztp $(date +"%b %d %H:%M:%S") ###" > $YUM_PUPPET
   echo -ne "[puppetlabs]\nname=puppetlabs\nenabled=1\ngpgcheck=1\n" >> $YUM_PUPPET
   echo "baseurl=$YUM_REPO" >> $YUM_PUPPET
   echo "gpgkey=$YUM_REPO/RPM-GPG-KEY-puppetlabs" >> $YUM_PUPPET 
}
function install_puppet(){
   # Install puppet 
   /usr/bin/yum clean all > /dev/null
   /usr/bin/yum update > /dev/null
   /usr/bin/yum install -y puppet > /dev/null
}
function config_puppet(){
  echo "### created by ztp $(date +"%b %d %H:%M:%S") ###" > $PUPPET_CONF
  echo -ne "[main]\nserver = $PUPPET_SRV\n" >> $PUPPET_CONF
}
function setup_resolver(){
  local resolver=/etc/resolv.conf
  echo "### created by ztp $(date +"%b %d %H:%M:%S") ###" > $resolver
  echo "domain $DOMAIN" >> $resolver
  echo "search $DOMAIN" >> $resolver
  echo "nameserver $DOMAIN_SRV" >> $resolver  
}
function set_hostname(){
  xrapply_string_with_reason "ztp puppet install" "hostname $HOSTNAME\n interface mgmtEth 0/RP0/CPU0/0\n ipv4 address $MGMT_IP \n"
  /bin/hostname -f $HOSTNAME.$DOMAIN
  echo $HOSTNAME > /etc/hostname  
}
function start_services(){
  /etc/init.d/sshd_operns start
  /etc/init.d/puppet start
}

### script start
set_hostname;
setup_resolver;
create_repo;
install_puppet;
config_puppet;
start_services;
exit 0

```
