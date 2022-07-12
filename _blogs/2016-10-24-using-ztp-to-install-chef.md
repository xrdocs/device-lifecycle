---
published: true
date: '2016-10-24 22:24 -0700'
title: Using ZTP to install Chef
author: Patrick Warichet
excerpt: Installing Chef with Zero Touch Provisioning
tags:
  - iosxr
  - cisco
---

{% include toc icon="table" title="Using ZTP to install chef" %}

## Introduction
Chef is an automation platform that "turns infrastructure into code", allowing users to manage and deploy resources across multiple servers, or nodes. Chef allows users to create and download recipes (stored in cookbooks) to automate content, configuration and policies on these nodes.

Chef is comprised of a Chef server, one or more workstations, and a number of nodes that are managed by the chef-client installed on each node. You can download the [IOS-XR Chef client package](https://packages.chef.io/stable/ios_xr/6/chef-12.15.19-1.ios_xr6.x86_64.rpm "IOS-XR Chef client package") from Chef and installed directly inside the control plane LXC of IOS-XR.

Chef like Puppet (see my previous blog [Using ZTP to install Puppet](https://xrdocs.github.io/software-management/blogs/2016-10-20-using-device-lifecycle-to-install-puppet/ "Using ZTP to install Puppet"))

Automated configuration management tools play a vital role in managing complex enterprise infrastructures. Amongst the many advantages, the ones pertinent to IOS-XR and network node in general are:

**Consistency:** It makes it easier for configuration changes to meet compliance and security requirements. By automating repeated tasks (like applying a SMU), it allows network administrators to concentrate on more important stuff. 

**Efficient change management:**. Automated configuration management can remove delay when deploying new technologies, reducing the number of processes needed to manage change. Small change batches can be performed on a more regular basis.

**Availability:** Automatated configuration management tool help quickly restore service. Rather than troubleshooting an issue by hand, a system can be reset to well known working status. 

**Visibility:** Configuration management tools include auditing and reporting capabilities, changes can be automatically logged in all relevant tracking systems.

## Chef Infrastructure
Chef requires the follwing components a server, one or more workastation and one or more nodes, The instruction below used Ubuntu Xenial for both the server and the workstation to mange the IOS-XR nodes.

### Installing the Chef Server

The Chef server is the central place that govern interaction between all workstations and managed nodes. Changes made in the workstations are uploaded to the Chef server, which is then accessed by the chef-client and used to configure individual nodes.
Installing Chef Server is easy as 1-2-3:

1 Download the latest [Chef Server]( http://downloads.chef.io/chef-server/.) for your favorite distro:
Example for Ubuntu Xenial (Chef version 12.9.1) 

```
wget https://packages.chef.io/stable/ubuntu/16.04/chef-server-core_12.9.1-1_amd64.deb
```
2 Install the server:

```
sudo dpkg -i chef-server-core_*.deb
```
3 Run the chef-server-ctl command to start the Chef server services:

```
sudo chef-server-ctl reconfigure
```

### Create a User and Organization

1 In order to link workstations and nodes to the Chef server, an administrator and an organization need to be created with associated RSA private keys. create a directory to store the keys:

```
mkdir ~/chef-keys
```
2 Create an administrator. Change username to your desired username, firstname and lastname to your first and last name, email to your email, password to a secure password, and username.pem to your username followed by .pem:

```
sudo chef-server-ctl user-create username firstname lastname email password --filename ~/chef-keys/username.pem
```
3 Create an organization. The shortname value should be a basic identifier for your organization with no spaces, whereas the fullname can be the full, proper name of the organization. The association_user value username refers to the username made in the step above:

```
sudo chef-server-ctl org-create shortname fullname --association_user username --filename ~/chef-keys/shortname.pem
```
With the Chef server installed and the RSA keys generated, you can move on to configuring your workstation, where all major work will be performed for your Chef’s nodes.

### Installing and Setting up the Chef Workstation

Your Chef workstation will be where you create and configure any recipes, cookbooks, attributes, and other changes made to your Chef configurations. Although this can be the same machine that host the server, it is recommended to keep the server and the workstation seperated.

1 Download the latest [Chef Development Kit](https://downloads.chef.io/chef-dk/ "Chef Development Kit"):

```
wget https://packages.chef.io/stable/ubuntu/12.04/chefdk_0.19.6-1_amd64.deb
```
2 Install ChefDK:

```
sudo dpkg -i chefdk_*.deb
```
3 Verify the components of the development kit:

```
~$ chef verify
Running verification for component 'berkshelf'
Running verification for component 'test-kitchen'
Running verification for component 'tk-policyfile-provisioner'
Running verification for component 'chef-client'
Running verification for component 'chef-dk'
Running verification for component 'chef-provisioning'
Running verification for component 'chefspec'
Running verification for component 'generated-cookbooks-pass-chefspec'
Running verification for component 'rubocop'
Running verification for component 'fauxhai'
Running verification for component 'knife-spork'
Running verification for component 'kitchen-vagrant'
Running verification for component 'package installation'
Running verification for component 'openssl'
Running verification for component 'inspec'
Running verification for component 'delivery-cli'
Running verification for component 'git'
Running verification for component 'opscode-pushy-client'
Running verification for component 'chef-sugar'
.................
---------------------------------------------
Verification of component 'test-kitchen' succeeded.
Verification of component 'chef-dk' succeeded.
Verification of component 'chefspec' succeeded.
Verification of component 'rubocop' succeeded.
Verification of component 'knife-spork' succeeded.
Verification of component 'openssl' succeeded.
Verification of component 'delivery-cli' succeeded.
Verification of component 'opscode-pushy-client' succeeded.
Verification of component 'berkshelf' succeeded.
Verification of component 'fauxhai' succeeded.
Verification of component 'inspec' succeeded.
Verification of component 'chef-sugar' succeeded.
Verification of component 'tk-policyfile-provisioner' succeeded.
Verification of component 'chef-provisioning' succeeded.
Verification of component 'kitchen-vagrant' succeeded.
Verification of component 'git' succeeded.
Verification of component 'chef-client' succeeded.
Verification of component 'package installation' succeeded.
Verification of component 'generated-cookbooks-pass-chefspec' succeeded.

```

4 Generate the chef-repo and add the RSA keys

```
~$ chef generate repo chef-repo
~$ cd chef-repo
~/chef-repo$ mkdir .chef
~/chef-repo$ scp user@chef-server:~/chef-keys/*.pem .chef/
```

5 Generate knife.rb

Using your favorite text editor create a knife configuration file named knife.rb in to your ~/chef-repo/.chef folder.

```
log_level                :info
log_location             STDOUT
node_name                'username'
client_key               '/home/cisco/chef-repo/.chef/username.pem'
validation_client_name   'shortname-validator'
validation_key           '/home/cisco/chef-repo/.chef/shortname.pem'
chef_server_url          'https://chef-server/organizations/shortname'
syntax_check_cache_path  '/home/cisco/chef-repo/.chef/syntax_check_cache'
cookbook_path [ '/home/cisco/chef-repo/cookbooks' ]
```
Replace username,shortname with the values used in the steps "Create a User and Organization"
Move uo to the chef-repo and copy the needed SSL certificates from the server:

```
~/chef-repo/.chef$ cd ..
~/chef-repo$ knife ssl fetch
WARNING: Certificates from chef-cook will be fetched and placed in your trusted_cert
directory (~/chef-repo/.chef/trusted_certs).

Knife has no means to verify these are the correct certificates. You should
verify the authenticity of these certificates after downloading.

Adding certificate for chef-cook in ~/chef-repo/.chef/trusted_certs/chef-cook.crt

```
Confirm that knife.rb is set up correctly by running the client list:

```
~/chef-repo$ knife client list
ciscolab-validator
```
This command should output the validator name (ciscolab in our case).
With both the server and a workstation configured, it is possible to bootstrap your first node.

## Installing the client with ZTP and Bootsrap the node 
Using ZTP we can install the Chef client directly inside the control plane LXC of IOS-XR, here is a script example that will perform the installation during the initial bootup.

```shell
#!/bin/bash

YUM_REPO="http://172.30.0.22/packages/chef"
YUM_CHEF="/etc/yum/repos.d/chef.repo"
CHEF_SRV="chef-cook.cisco.local"
DOMAIN="cisco.local"
DOMAIN_SRV=172.30.0.25
HOSTNAME="ncs-5001-c"
MGMT_IP="172.30.12.54 255.255.255.0"

source ztp_helper.sh

function create_repo(){
   # Create local repository file for chef
   echo "creating repo file in /etc/yum/repo.d"
   echo "### created by ztp $(date +"%b %d %H:%M:%S") ###" > $YUM_CHEF
   echo -ne "[chef]\nname=chef\nenabled=1\ngpgcheck=1\n" >> $YUM_CHEF
   echo "baseurl=$YUM_REPO" >> $YUM_CHEF
   echo "gpgkey=$YUM_REPO/chef.asc" >> $YUM_CHEF 
}
function install_chef(){
   # Install chef from local repository
   echo "installing chef from the local repo"
   /usr/bin/yum clean all > /dev/null
   /usr/bin/yum update > /dev/null
   /usr/bin/yum install -y chef > /dev/null
}
function setup_resolver(){
  echo " setting up the resolver"
  local resolver=/etc/resolv.conf
  echo "### created by ztp $(date +"%b %d %H:%M:%S") ###" > $resolver
  echo "domain $DOMAIN" >> $resolver
  echo "search $DOMAIN" >> $resolver
  echo "nameserver $DOMAIN_SRV" >> $resolver  
}
function set_hostname(){
  echo "setting up the device hostname"
  xrapply_string_with_reason "ztp chef install" "hostname $HOSTNAME\n interface mgmtEth 0/RP0/CPU0/0\n ipv4 address $MGMT_IP \n"
  /bin/hostname -f $HOSTNAME.$DOMAIN
  echo $HOSTNAME > /etc/hostname  
}
function start_services(){
  echo "starying services"
  /etc/init.d/sshd_operns start
  # Start chef client in daemon mode and schedule a run every 5 min
  /usr/bin/chef-client -daemonize -i 300 -L /var/log/chef.log
}

### script start
set_hostname;
setup_resolver;
create_repo;
install_chef;
start_services;

exit 0
```

On the workstation, we bootstrap the node using knife. It is important to note that the Chef client will run inside the Linux shell. Inside the Linux shell the default port for the ssh server is 57722 and ssh using the root user is disabled. Fortunatly IOS-XR root-lr users are not root when they ssh into the system (see my previous blog [IOS-XR  Users and Groups](https://xrdocs.github.io/software-management/blogs/2016-10-17-ios-xr-users-and-groups-inside-linux/ "IOS-XR Users and Groups")). Since we need root access to bootstrap the client we have to use --sudo and provide the user password.

```
~$ knife bootstrap 172.30.12.54 --sudo -p 57722 -x admin -P cisco123 --node-name ncs-5001-c
Connecting to 172.30.12.54
172.30.12.54 knife sudo password: 
Enter your password: 
172.30.12.54 
172.30.12.54 -----> Existing Chef installation detected
172.30.12.54 Starting the first Chef Client run...
172.30.12.54 Starting Chef Client, version 12.15.19
172.30.12.54 [2016-11-02T23:09:51+00:00] WARN: [inet] no ip address on fwdintf
172.30.12.54 Creating a new client identity for ncs-5001-c using the validator key.
172.30.12.54 resolving cookbooks for run list: []
172.30.12.54 Synchronizing Cookbooks:
172.30.12.54 Installing Cookbook Gems:
172.30.12.54 Compiling Cookbooks...
172.30.12.54 [2016-11-02T23:09:53+00:00] WARN: Node ncs-5001-c has an empty run list.
172.30.12.54 Converging 0 resources
172.30.12.54 
172.30.12.54 Running handlers:
172.30.12.54 Running handlers complete
172.30.12.54 Chef Client finished, 0/0 resources updated in 04 seconds

knife node list
ncs-5001-c
```

## Creating a simple recipe

We use the command "chef generate cookbook" to configure our cookbook, once created, we change to the recipes folder and edit the default.rb recipe. 

```
~/chef-repo/cookbooks$ chef generate cookbook ios-xr
Generating cookbook ios-xr
- Ensuring correct cookbook file content
- Ensuring delivery configuration
- Ensuring correct delivery build cookbook content

Your cookbook is ready. Type `cd ios-xr` to enter it.

There are several commands you can run to get started locally developing and testing your cookbook.
Type `delivery local --help` to see a full list.

Why not start by writing a test? Tests for the default recipe are stored at:

test/recipes/default_test.rb

If you'd prefer to dive right in, the default recipe can be found at:

recipes/default.rb

```
We create a simple recipe that will create a file in the home directory of the user (/disk0: for IOS-XR)

```
~/chef-repo/cookbooks$ cd ios-xr/recipes/
~/chef-repo/cookbooks$ vi default.rb
#
# Cookbook Name:: ios-xr
# Recipe:: default
#
# Copyright 2016, YOUR_COMPANY_NAME
#
# All rights reserved - Do Not Redistribute
#
file "#{ENV['HOME']}/chef.txt" do
  content 'Hello from Chef'
end
```

We push the recipe to the Chef server and add it to the run list for the ncs-5001-c node

```
~/chef-repo/cookbooks/ios-xr/recipes$ knife cookbook upload ios-xr
Uploading ios-xr         [0.1.0]
Uploaded 1 cookbook.

~/chef-repo/cookbooks/ios-xr$ knife node run_list set ncs-5001-c 'recipe[ios-xr::default]'
ncs-5001-c:
  run_list: recipe[ios-xr::default]
```

The execution will occur within the interval configured (300 sec in our case), Here is what the logs look like:

```
Starting Chef Client, version 12.15.19
resolving cookbooks for run list: ["ios-xr::default"]
Synchronizing Cookbooks:
  - ios-xr (0.1.0)
Installing Cookbook Gems:
Compiling Cookbooks...
Converging 1 resources
Recipe: ios-xr::default
  * file[/disk0:/chef.txt] action create
    - create new file /disk0:/chef.txt
    - update content in file /disk0:/chef.txt from none to ba4fda
    --- /disk0:/chef.txt	2016-11-09 22:04:29.356188978 +0000
    +++ /disk0:/.chef-chef20161109-16571-14v6aep.txt	2016-11-09 22:04:29.355188978 +0000
    @@ -1 +1,2 @@
    +Hello from Chef

Running handlers:
Running handlers complete
Chef Client finished, 1/1 resources updated in 03 seconds
```
