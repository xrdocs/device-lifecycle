---
published: true
date: '2016-08-26 10:00 -0700'
title: Working with Zero Touch Provisioning
author: Patrick Warichet
excerpt: A brief introduction to Zero Touch Provisioning
tags:
  - iosxr
  - cisco
---
{% include toc icon="table" title="IOS-XR: Working with ZTP" %}

## Purpose of ZTP
ZTP was designed to perform 2 different operations:

1. Download and apply an initial configuration.
2. Download and execute a shell script.

## How ZTP works
The ZTP process is executed or invoked inside the control plane LXC Linux shell. Prior to IOS-XR 6.1.1 ZTP was executed within the default network namespace and could not access directly the data interfaces. Starting with IOS-XR 6.1.1, ZTP is executed inside the global-VRF network namespace with full access to all the data interfaces. This document is based on the IOS-XR 6.1.1 implementation.
ZTP is launched from XR process manager (processmgr) when the system reaches level 999 (last processes to be scheduled for execution). At the beginning of its execution, ZTP will scan the configuration for the presence of a username. If there are no username configured, ZTP will fork a DHCP client on the management interface for IPv4 and IPv6 simultaneously, and wait for a response.

If the DHCP response contains an option 67 (option 59 for IPv6), ZTP will download the file using the URI provided by option 67 (or option 59 for IPv6).
If the file received is not a text file or the file is larger than 100 MB ZTP will erase the file and terminate its execution.
Otherwise it will analyze the first line of the text file received, if the first line of the file starts with **"!! IOS XR"**, it will consider it as a configuration file and pass it to the command line interpreter for syntax verification and commit it.
If the first line starts with **"#!/bin/bash"** or **"#!/bin/sh"** ZTP will assume this is a script and start the execution, as illustrate below.
If the text file cannot be interpreted as a shell script or a configuration file, ZTP will erase the file and terminate its execution.
The script can use all the Linux tools available in the Control Plane LXC and perform addition HTTP GET using wget or curl for example to install package and/or download and apply configuration blocks. 
![ztp-flow.png]({{site.baseurl}}/images/ztp-flow.png "ZTP operation flow")

---
**Note:** In IOS-XR release 6.1.1 ZTP can also be invoked from the command line interpreter in this case it will start its execution even if a username or a configuration is present in the system. 

---

## ZTP requirement
ZTP requires 2 external services: a DHCP server and an HTTP server, as illustrated above, the support for tftp has been dropped for security and reliability reasons.

### DHCP server configuration
The basic configuration described below provides a fixed IP address and a configuration file taking in account only the mac address of the management interface.

```
host ncs-5001-rp0 {
   hardware ethernet e4:c7:22:be:10:ba;
   fixed-address 172.30.12.54;
   filename "http://172.30.0.22/configs/ncs5001-1.config";
}
```

A more elaborate example that takes into account option 77 or option 15 for IPv6 (user-class) embedded in the dhcp request sent by the client, ZTP embed the string “exr-config” in the DHCP request as described below. The if statement also take into account the capability to re-image the system using iPXE (see [iPXE deep dive](https://xrdocs.github.io/software-management/tutorials/2016-07-27-ipxe-deep-dive/ "iPXE deep dive") document)

```
host ncs-5001-rp0 {
   hardware ethernet e4:c7:22:be:10:ba;
   fixed-address 172.30.12.54;
   if exists user-class and option user-class = "iPXE" {
      filename = "http://172.30.0.22/boot.ipxe";
   } elsif exists user-class and option user-class = "exr-config" {
      filename = "http://172.30.0.22/scripts/ncs-5001-rp0_ztp.sh";
   }
}
```

Since ZTP does not require any intervention on the system, an easier way to provision the system is to use the serial number printed on the box and/or the RP, the configuration is then as follow:

```
host ncs-5001-rp0 {
   option dhcp-client-identifier "FOC1947R144";
   fixed-address 172.30.12.54;
   if exists user-class and option user-class = "iPXE" {
      filename = "http://172.30.0.22/boot.ipxe";
   } elsif exists user-class and option user-class = "exr-config" {
      filename = "http://172.30.0.22/scripts/ncs-5001-rp0_ztp.sh";
   }
}
```

---
**Note:** The DHCP configuration examples described in this document use the open-source isc-dhcp-server syntax.

---

### HTTP Server requirement
The HTTP server should be reachable from the management interface or from a data interface if invoked manually and should be readable.

## ZTP utilities
ZTP includes a set of CLI commands and a set of shell utilities that can be sourced within the user script.

### ztp_helper.sh
ztp_helper.sh is a shell script that can be sourced by the user script, it provides simple utilities to access some XR functionalities. These utilities are: 

**xrcmd:** Runs an XR exec command

```
xrcmd “show running”
```

**xrapply:** Applies the block of configuration, specified in a file:

```
cat >/tmp/config <<EOF
!! XR config example
hostname pluto
EOF
xrapply /tmp/config
```

**xrapply_with_reason:** Same as above, but specify a reason for commit history tracking

```
cat >/tmp/config <<EOF
!! XR config example
hostname saturn
EOF
xrapply_with_reason "this is an important name change" /tmp/config
```

**xrapply_string:** Applies a block of configuration specified in a string. Use "\n" to delimit line of configuration statement.

```
xrapply_string "hostname mars\n interface TenGigE0/0/0/0\n ipv4 address 172.30.0.144/24\n”
```

**xrapply_string_with_reason:** As above, but specify a reason for commit history tracking:

```
xrapply_string_with_reason ”system renamed again" "hostname venus\n interface TenGigE0/0/0/0\n ipv4 address 172.30.0.144/24\n”
```

## ZTP CLI Commands
**ztp initiate:** Invokes a new ZTP DHCP session, logs will go to the console and /disk0:/ztp/ztp.log

ztp initiate allows the execution of a script even of the system has already been configured. This command is useful for testing ZTP without forcing a reload. This command is particularly useful to test scripts or if some manual operations are required before provisioning the box. With ZTP initiate you can specify any data interfaces on the system to be used for the whole ZTP process.

```
RP/0/RP0/CPU0:venus#ztp initiate debug verbose interface TenGigE 0/0/0/0
Invoke ZTP? (this may change your configuration) [confirm] [y/n] :
```

**ztp terminate:** Terminates any ZTP session in progress

```
RP/0/RP0/CPU0:venus#ztp terminate verbose
Mon Oct 10 16:52:38.507 UTC
Terminate ZTP? (this may leave your system in a partially configured state) [confirm] [y/n] :y
ZTP terminated
```

**ztp breakout:** Performs a 4x10 breakout detection on all 40 Gig interfaces, by default if no link is detected on any of the four 10Gig interfaces, the port will remain in 40 Gig mode.
The subcommand nosignal-stay-in-breakout-mode will force the port in breakout mode even if there is no link signal detected but will place the interfaces in shutdown mode. The subcommand nosignal-stay-in-state-noshut will leave the port in breakout mode but will place the four 10Gig interface in no shutdown mode.
The command "ztp breakout" may not be supported on the ASR9K routers.

```
RP/0/RP0/CPU0:venus#ztp breakout ?
apply  XR configuration commands to apply(cisco-support)
debug  Run with additional logging to the console(cisco-support)
hostname  XR hostname to set(cisco-support)
nosignal-stay-in-breakout-mode On no signal, prefer interfaces to remain in breakout mode(cisco-support)
nosignal-stay-in-state-noshut  On no signal, prefer interfaces to be noshut(cisco-support)
verbose  Run with logging to the console(cisco-support)                           
```

**ztp clean:** Remove all ZTP files saved on disk

```
RP/0/RP0/CPU0:venus#ztp clean verbose
Mon Oct 10 17:03:43.581 UTC
Remove all ZTP temporary files and logs? [confirm] [y/n] :y
All ZTP files have been removed.
If you now wish ZTP to run again from boot, do 'conf t/commit replace' followed by reload.
```

## ZTP Logging
ZTP logs its operation on the flash file system in the directory /disk0:/ztp/. ZTP logs all the transaction with the DHCP server and all the state transition. Prior executions of ZTP are also logged in /disk0:/ztp/old_logs/

### Simple ZTP Example
In the following example we will review the execution of a simple configuration script downloaded from a data interface using the command “ztp initiate interface Ten 0/0/0/0 verbose”, this script will unshut all the interfaces of the system and configure a load interval of 30 sec on all of them.

```bash
#!/bin/bash
#############################################################################
# *** Be careful this is powerful and can potentially destroy your system ***
#                *** !!! Use at your own risk !!! ***
#
# Script file should be saved on the backend HTTP server
#
# Tested on NCS-5501 with IOS-XR 6.1.1
#
#############################################################################

source /pkg/bin/ztp_helper.sh
config_file="/tmp/config.txt"
interfaces=$(xrcmd "show interfaces brief")

function activate_all_if(){
  arInt=($(echo $interfaces | grep -oE '(Te|Fo|Hu)[0-9]*/[0-9]*/[0-9]*/[0-9]*'))
  for int in ${arInt[*]}; do
    echo -ne "interface $int\n no shutdown\n load-interval 30\n" >> $config_file
  done
  xrapply_with_reason "Initial ZTP configuration" $config_file
}

### Script entry point
if [ -f $config_file ]; then
  /bin/rm -f $config_file
else
  /bin/touch $config_file
fi
activate_all_if;
exit 0
```

### More Complex Example
In this example, the HTTP server hosts a CSV file that contains devices serial number followed by the hostname. The HTTP server also contains a basic configuration file that need to be applied. Finally a local repository accessible by HTTP contains IOS-XR and third party packages to be installed.

After bootup ZTP will provide its serial number and query the back-end database using HTTP POST, once it obtains its hostname it will perform a version check, if the version on the system does not match the desired version, the system will change the boot order forcing a reboot using iPXE that will hopefully re-image the system to the desired version.

If the system is running the correct version the script proceed by installing the k9sec package and create a generic RSA key for SSH. It will then add a local repository for third party packages and install midnight commander and its dependent packages. The script finishes its execution after downloading and applying the configuration.

**ZTP Script** ncs-5001-rp0_ztp.sh

```bash
#!/bin/bash
#############################################################################
# *** Be careful this is powerful and can potentially destroy your system ***
#                *** !!! Use at your own risk !!! ***
#
# Script file should be saved on the backend HTTP server
#
# Tested on NCS-5501 with IOS-XR 6.1.1
#
#############################################################################
export LOGFILE=/disk0:/ztp/user-script.log
export HTTP_SERVER=http://172.30.0.22
export SYSLOG_SERVER=172.30.0.22
export SYSLOG_PORT=514
export CONFIG_PATH=configs
export SCRIPT_PATH=scripts
export RPM_PATH="packages/ncs5k/6.1.1"
export PHP_SCRIPT="php/device_name.php"
export DESIRED_VER="6.1.1"
K9SEC_RPM=ncs5k-k9sec-3.1.0.0-r611.x86_64.rpm

## ztp_helper is inside the Fretta Code-base - ASSUMPTION
source ztp_helper.sh

function ztp_log() {
    # Sends logging information to local file and syslog server
    syslog "$1"
    echo "$(date +"%b %d %H:%M:%S") "$1 >> $LOGFILE
}

function syslog() {
    # Sends syslog messages with netcat (nc)
  echo "ztp-script: "$1 | nc -u -q1 $SYSLOG_SERVER $SYSLOG_PORT
}

function get_hostname(){
    # Uses serial number to query a remote database and get hostname using HTTP POST
    local sn=$(dmidecode | grep -m 1 "Serial Number:" | awk '{print $NF}');
    local result="`wget -O- --post-data="serial=$sn" ${HTTP_SERVER}/${PHP_SCRIPT}`";
    if [ "$result" != "Not found" ]; then
            DEVNAME=$result;
            return 0
    else
    	ztp_log "Serial $sn not found, hostname not set";
        return 1
    fi
}

function download_config(){
	# Downloads config using hostname
    ztp_log "### Downloading system configuration ###";
    /usr/bin/wget ${HTTP_SERVER}/${CONFIG_PATH}/${DEVNAME}.config -O /disk0:/new-config 2>&1 >> $LOGFILE
    if [[ "$?" != 0 ]]; then
    	ztp_log "### Error downloading system configuration ###"
    else
        ztp_log "### Downloading system configuration complete ###";
    fi
}

function apply_config(){
	# Applies initial configuration
    ztp_log "### Applying initial system configuration ###";
    xrapply_with_reason "Initial ZTP configuration" /disk0:/new-config 2>&1 >> $LOGFILE;
    ztp_log "### Checking for errors ###";
    local config_status=$(xrcmd "show configuration failed");
    if [[ $config_status ]]; then
    	echo $config_status  >> $LOGFILE
        ztp_log "!!! Error encounter applying configuration file, review the log !!!!";
    fi
    ztp_log "### Applying system configuration complete ###";
}

function install_k9sec_pkg(){
    # Installs the k9sec package from repository, create a RSA key modulus 1024
    ztp_log "### XR K9SEC INSTALL ###"
    /usr/bin/wget ${HTTP_SERVER}/${RPM_PATH}/${K9SEC_RPM} -O /disk0:/$K9SEC_RPM 2>&1
    if [[ "$?" != 0 ]]; then
        ztp_log "### Error downloading $K9SEC_RPM ###"
    else
        ztp_log "### Downloading $K9SEC_PKG complete ###";
    fi
  xrcmd "install update source /disk0:/ $K9SEC_RPM" 2>&1 >> $LOGFILE
  local complete=0
  while [ "$complete" = 0 ]; do
        complete=`xrcmd "show install active" | grep k9sec | head -n1 | wc -l`
        ztp_log "Waiting for k9sec package to be activated"
        sleep 5
    done
    if [[ -z $(xrcmd "show crypto key mypubkey rsa") ]]; then
        echo "1024" | xrcmd "crypto key generate rsa"
    else
        echo -ne "yes\n 1024\n" | xrcmd "crypto key generate rsa"
    fi
    rm -f /disk0:/$K9SEC_RPM
    ztp_log "### XR K9SEC INSTALL COMPLETE ###"
}

function check_version(){
	# returns 0 is version matches, 1 otherwise
    local current_ver=`xrcmd "show version" | grep Version | grep Cisco | cut -d " " -f 6`;
    ztp_log "current=$current_ver, desired=$DESIRED_VER";
    if [ "$DESIRED_VER" = "$current_version" ]; then
        return 0
    else
        return 1
    fi
}

function reboot_ipxe(){
    # Do not use in production, may not be supported on NCS-5508 
    ztp_log "### Mounting EFIvar and changing boot order"
    local EFI_FILESYS="/sys/firmware/efi/efivars"
    if [ ! -d $EFI_FILESYS ]; then
        ztp_log "EFI mount point not present"
    fi 
    /bin/mount -t efivarfs efivarfs $EFI_FILESYS
    if [[ "$?" != 0 ]]; then
    ztp_log "Error mounting efivars filesystem";
  fi
  local iPXE=$(/usr/sbin/efibootmgr | grep IPXE | awk -v FS="(Boot|*)" '{print $2}')
  local MMC=$(/usr/sbin/efibootmgr | grep "HS-SD/MMC" | awk -v FS="(Boot|*)" '{print $2}')
  local Shell=$(/usr/sbin/efibootmgr | grep Shell | awk -v FS="(Boot|*)" '{print $2}')
    /usr/sbin/efibootmgr -o $iPXE,$MMC,$Shell
    if [[ "$?" != 0 ]]; then
    ztp_log "Error changing boot order";
  fi
    ztp_log "### Resetting the system";
    echo 1 > /proc/sys/kernel/sysrq 
    echo b > /proc/sysrq-trigger
    ztp_log "Unable to reset the system";
}

function add_repo(){
	# add a local repo to yum package manager
    /usr/bin/yum-config-manager --add-repo ${HTTP_SERVER}/${RPM_PATH} 2>&1
}

function install_mc(){
	# uses yum to install packages and dependant package(s) automatically
    if /usr/bin/yum list installed "mc" >/dev/null 2>&1; then
    	ztp_log "### Installing midnight commander";
        /usr/bin/yum install mc -y 2>&1
    else
    	ztp_log "### Midnight commander already installed ###"
    fi
}

# ==== Script entry point ====
get_hostname;
if [[ "$?" != 0 ]]; then
    ztp_log "No valid hostname found terminating ZTP";
    exit 1
fi
ztp_log "Hello from ${DEVNAME} !!!";
check_version;
if  [[ "$?" != 0 ]]; then
    ztp_log "Version mismatch, we will upgrade using iPXE";
    reboot_ipxe;
else
    ztp_log "Version match, proceeding to configuration";
fi
ztp_log "Starting autoprovision process...";
install_k9sec_pkg;
add_repo;
install_mc;
download_config;
apply_config;
ztp_log "Autoprovision complete...";
exit 0
```

**Backend PHP script** device_name.php

```php
<?php
$file = 'inventory.txt';
$searchfor = ($_POST['serial']);
// the following line prevents the browser from parsing this as HTML.
header('Content-Type: text/plain');
// get the file contents, assuming the file to be readable (and exist)
$contents = file_get_contents($file);
// escape special characters in the query
$pattern = preg_quote($searchfor, '/');
// finalise the regular expression, matching the whole line
$pattern = "/^.*$pattern.*\$/m";
// search, and store first matching occurences in $matches
if(preg_match($pattern, $contents, $matches)){
   //match the last element of the  line
   preg_match('/[^,]*$/', $matches[0], $results);
   echo $results[0];
}
else{
   echo "Not found";
}
?>
```

**CSV file** inventory.txt

```
FOC2647D246,ncs-5001-a
FOC1568P682,ncs-5001-b
FOC1947R143,ncs-5001-c
```

**Logging output**

```
Oct 11 11:05:38 172.30.0.54 ztp-script: Hello from ncs-5001-c !!!
Oct 11 11:05:40 172.30.0.54 ztp-script: current=6.1.1, desired=6.1.1
Oct 11 11:05:40 172.30.0.54 ztp-script: Version match, proceeding to configuration
Oct 11 11:05:41 172.30.0.54 ztp-script: Starting autoprovision process...
Oct 11 11:05:42 172.30.0.54 ztp-script: ### XR K9SEC INSTALL ###
Oct 11 11:05:44 172.30.0.54 ztp-script: ### Downloading complete ###
Oct 11 11:05:55 172.30.0.54 ztp-script: Waiting for k9sec package to be activated
Oct 11 11:06:01 172.30.0.54 ztp-script: ### XR K9SEC INSTALL COMPLETE ###
Oct 11 11:06:03 172.30.0.54 ztp-script: ### Installing midnight commander ###
Oct 11 11:06:04 172.30.0.54 ztp-script: ### Downloading system configuration ###
Oct 11 11:06:05 172.30.0.54 ztp-script: ### Downloading system configuration complete ###
Oct 11 11:06:06 172.30.0.54 ztp-script: ### Applying initial system configuration ###
Oct 11 11:06:11 172.30.0.54 ztp-script: !!! Checking for errors !!!
Oct 11 11:06:14 172.30.0.54 ztp-script: ### Applying system configuration complete ###
Oct 11 11:06:15 172.30.0.54 ztp-script: Autoprovision complete...
```
