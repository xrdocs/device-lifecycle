---
published: true
date: '2017-10-25 12:50 -0700'
title: >-
  IOS-XR ZTP Python Library: Automated provisioning using python (6.2.25+) -
  Part 1 
author: Akshat Sharma
excerpt: >-
  Understand the Python ZTP library in IOS-XR, the available methods and build
  your own python based provisioning scripts for IOS-XR platforms.
position: hidden
tags:
  - iosxr
  - cisco
  - linux
  - ztp
  - python
  - CLI
  - automate
  - provision
  - dhcp
---

{% include toc icon="table" title="IOS-XR ZTP Python" %}
{% include base_path %}

## Introduction

If you've not had a chance to play around with Zero Touch Provisioning (ZTP) on IOS-XR, then I would implore you to take a look at the following blogs:

*  **IOS-XR ZTP: Learning through Packet Captures**: Akshat Sharma tells you everything you need to know to understand the DHCP (v4 and v6) server setup and packet captures to illustrate the DHCP options that may be used for classification of devices during provisioning:
><https://xrdocs.github.io/software-management/blogs/2017-09-21-ios-xr-ztp-learning-through-packet-captures/>

*  **Working with ZTP (bash scripts)**: Patrick Warichet walks through the ZTP workflow and illustrates examples of bash scripts for zero touch provisioning using the ZTP bash utilities that enable the bash script to interact with IOS-XR CLI.
><https://xrdocs.github.io/software-management/tutorials/2016-08-26-working-with-ztp/>

Apart from a new DHCPv4 option introduced in IOS-XR Release 6.2.25 (option 124), both the articles above are valid for IOS-XR starting from release 6.1.2.


In **Part 1** of this tutorial we will introduce a new ZTP library released as part of IOS-XR 6.2.25 that enables users to write ZTP scripts directly in python without having to create wrappers around the native bash utilities (The bash utilities were introduced in XR Release 6.1.2 and are explained here : [ZTP bash utilties](https://xrdocs.github.io/software-management/tutorials/2016-08-26-working-with-ztp/#ztp-utilities)). We will finish this tutorial with a simple python ZTP script that will create a root user in XR and exercise some of the methods available in the library. 

In **Part 2** of this tutorial, we'll go a step further and write a much more exhaustive ZTP script, in python, that handles:
  * Installation of packages in XR : k9sec and mgbl RPMs
  * IOS-XR config replace
  * Spin up a docker container 
  * install puppet
  * Set up a cron job
  * All on a dual RP NCS5500 - so the ZTP script will do initial checkpointing of linux packages and files for us!


**Let's get started!**
{: .notice--success}


## NCS5508 ZTP setup


We will be using an NCS5508 router running IOS-XR 6.2.25 CCO image. Bear in mind that the script we write will work perfectly fine on xrv9k (virtual IOS-XR) as well. This tutorial focuses on NCS5508 in order to have a dual RP setup to play with, especially for Part 2.

The setup looks something like this:



## The ztp_helper.py Libary

The ZtpHelpers class is implemented in ztp_helpers.py  script. In this github repository you will find the library itself in the `/lib` directory.
This library is available on the router by default starting IOS-XR 6.2.2.
This file will exist at the location:  /pkg/bin/ztp_helper.py  on your router.

The python library is provided in the github repository to help a user easily understand the structure of the library
and then inherit in the user side ztp script.


To utilize the python library and/or inherit in user defined classes, make sure your syspath includes /pkg/bin as
shown below:

```
import sys
sys.path.append("/pkg/bin/")
from ztp_helper import ZtpHelpers

```

You will find sample scripts to get you started at the root of this github repository.
Specifically:

*  **sample_ztp_script.py** : This is a sample ZTP script that creates an inherited class and runs some sample commands to show how XR CLI commands, syslog, error checking woul
d work.

*  **exhaustive_ztp_script.py** :  This is a much more exhaustive script. It actually solves a ZTP use case where the mgmt port is placed in a VRF, we enable moving across netw
ork namespaces, set up cron jobs, work with active/standby RPs etc. Use this script to leverage some code pieces in your own script, if needed.


### Library Methods:

This class exposes the following methods and handles all the error handling:

*  **\_\_init\_\_(self, syslog_server=None, syslog_port=None, syslog_file=None)**:
```
            __init__ constructor
           :param syslog_server: IP address of reachable Syslog Server 
           :param syslog_port: Port for the reachable syslog server
           :param syslog_file: Alternative or addon file for syslog
           :type syslog_server: str
           :type syslog_port: int
           :type syslog_file:str

            All parameters are optional. When nothing is specified during object creation, then 
            all logs are sent to a log rotated file /tmp/ztp_python.log (max size of 1MB).
``` 

*  **setns(cls, fd, nstype)**:  

```
           Class Method for setting the network namespace
           :param cls: Reference to the class ZtpHelpers 
           :param fd: incoming file descriptor  
           :param nstype: namespace type for the sentns call
           :type nstype: int  
                  0      Allow any type of namespace to be joined.
                  CLONE_NEWNET = 0x40000000 (since Linux 3.0)
                         fd must refer to a network namespace.
                         
```

* **get_netns_path(cls, nspath=None, nsname=None, nspid=None)**: 

```
           Class Method to fetch the network namespace filepath 
           associated with a PID or name 
           :param cls: Reference to the class ZtpHelpers
           :param nspath: optional network namespace associated name
           :param nspid: optional network namespace associate PID
           :type nspath: str
           :type nspid: int 
           :return: Return the complete file path 
           :rtype:  str 
           
```

* **toggle_debug(self, enable)**:

```
            Enable/disable debug logging
           :param enable: Enable/Disable flag 
           :type enable: int


```

* **set_vrf(self, vrfname=None)**:

```
            Set the VRF (network namespace)
           :param vrfname: Network namespace name 
                           corresponding to XR VRF  
                           
```

*  **download_file(self, file_url, destination_folder)**:   

```
            Download a file from the specified URL
           :param file_url: Complete URL to download file 
           :param destination_folder: Folder to store the 
                                      downloaded file
           :type file_url: str
           :type destination_folder: str
           :return: Dictionary specifying download success/failure
                    Failure => { 'status' : 'error' }
                    Success => { 'status' : 'success',
                                 'filename' : 'Name of downloaded file',
                                 'folder' : 'Directory location of downloaded file'}
           :rtype: dict 
           
```

*  **setup_syslog(self)**:   

```
Method to Correctly set sysloghandler in the correct VRF (network namespace) and 
point to a remote syslog Server or local file or default log-rotated log file.
```

*  **xrcmd(self, cmd=None)**:  


```
            Issue an IOS-XR exec command and obtain the output
           :param cmd: Dictionary representing the XR exec cmd
                       and response to potential prompts
                       { 'exec_cmd': '', 'prompt_response': '' }
           :type cmd: dict            
           :return: Return a dictionary with status and output
                    { 'status': 'error/success', 'output': '' }
           :rtype: dict
```

*  **xrapply(self, filename=None, reason=None)**:  

```
           Apply Configuration to XR using a file 
           :param file: Filepath for a config file
                        with the following structure: 
                        !
                        XR config command
                        !
                        end
           
           :param reason: Reason for the config commit.
                          Will show up in the output of:
                          "show configuration commit list detail"
           :type filename: str
           :type reason: str
           :return: Dictionary specifying the effect of the config change
                     { 'status' : 'error/success', 'output': 'exec command based on status'}
                     In case of Error:  'output' = 'show configuration failed' 
                     In case of Success: 'output' = 'show configuration commit changes last 1'
           :rtype: dict 
 ```
 
*  **xrapply_string(self, cmd=None, reason=None)**:

```
            Apply Configuration to XR using  a single line string
           :param cmd: Single line string representing an XR config command  
           :param reason: Reason for the config commit.
                          Will show up in the output of:
                          "show configuration commit list detail"
           :type cmd: str
           :type reason: str 
           :return: Dictionary specifying the effect of the config change
                     { 'status' : 'error/success', 'output': 'exec command based on status'}
                     In case of Error:  'output' = 'show configuration failed'
                     In case of Success: 'output' = 'show configuration commit changes last 1'
           :rtype: dict
```

*  **xrreplace(self, filename=None)**:

```
           Replace XR Configuration using a file

           :param file: Filepath for a config file
                        with the following structure:

                        !
                        XR config commands
                        !
                        end
           :type filename: str
           :return: Dictionary specifying the effect of the config change
                     { 'status' : 'error/success', 'output': 'exec command based on status'}
                     In case of Error:  'output' = 'show configuration failed'
                     In case of Success: 'output' = 'show configuration commit changes last 1'
           :rtype: dict
```

### Sample Run Output
Checkout `sample_ztp_script.py` to see how to use the `ZtHelpers` Class and to write your own methods in the child class.
The output from `sample_ztp_script.py` run on IOS-XR shell when `ztp_helpers.py` is available in the `PYTHONPATH` is shown below:

```
[apple2:~]$ python sample_ztp_script.py 

###### Debugs enabled ######


###### Change context to user specified VRF ######


###### Using Child class method, setting the root user ######

2016-12-17 04:23:24,091 - DebugZTPLogger - DEBUG - Config File content to be applied  !
                     username netops
                     group root-lr
                     group cisco-support
                     secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1
                     !
                     end
2016-12-17 04:23:28,546 - DebugZTPLogger - DEBUG - Received exec command request: "show configuration commit changes last 1"
2016-12-17 04:23:28,546 - DebugZTPLogger - DEBUG - Response to any expected prompt ""
Building configuration...
2016-12-17 04:23:29,329 - DebugZTPLogger - DEBUG - Exec command output is ['!! IOS XR Configuration version = 6.2.1.21I', 'username netops', 'group root-lr', 'group cisco-support', 'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1', '!', 'end']
2016-12-17 04:23:29,330 - DebugZTPLogger - DEBUG - Config apply through file successful, last change = ['!! IOS XR Configuration version = 6.2.1.21I', 'username netops', 'group root-lr', 'group cisco-support', 'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1', '!', 'end']


###### Debugs Disabled ######


###### Executing a show command ######

Building configuration...
{'output': ['!! IOS XR Configuration version = 6.2.1.21I',
            '!! Last configuration change at Sat Dec 17 04:23:25 2016 by UNKNOWN',
            '!',
            'hostname customer2',
            'username root',
            'group root-lr',
            'group cisco-support',
            'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1',
            '!',
            'username noc',
            'group root-lr',
            'group cisco-support',
            'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1',
            '!',
            'username netops',
            'group root-lr',
            'group cisco-support',
            'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1',
            '!',
            'username netops2',
            'group root-lr',
            'group cisco-support',
            'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1',
            '!',
            'username netops3',
            'group root-lr',
            'group cisco-support',
            'secret 5 $1$7kTu$zjrgqbgW08vEXsYzUycXw1',
            '!',
            'cdp',
            'service cli interactive disable',
            'interface MgmtEth0/RP0/CPU0/0',
            'ipv4 address 11.11.11.59 255.255.255.0',
            '!',
            'interface TenGigE0/0/0/28/0',
            'shutdown',
            '!',
            'interface TenGigE0/0/0/28/3',
            'shutdown',
            '!',
            'controller Optics0/0/0/28',
            'breakout 4x10',
            '!',
            'interface FortyGigE0/0/0/30',
            'shutdown',
            '!',
            'interface preconfigure FortyGigE0/0/0/28',
            'shutdown',
            '!',
            'router static',
            'address-family ipv4 unicast',
            '0.0.0.0/0 11.11.11.2',
            '!',
            '!',
            'end'],
 'status': 'success'}

###### Apply valid configuration using a file ######

Building configuration...
{'status': 'success', 'output': ['!! IOS XR Configuration version = 6.2.1.21I', 'hostname customer', 'cdp', 'end']}

###### Apply valid configuration using a string ######

Building configuration...
{'output': ['!! IOS XR Configuration version = 6.2.1.21I',
            'hostname customer2',
            'end'],
 'status': 'success'}

###### Apply invalid configuration using a string ######

{'output': ['!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to',
            '!! one or more of the following reasons:',
            '!!  - the entered commands do not exist,',
            '!!  - the entered commands have errors in their syntax,',
            '!!  - the software packages containing the commands are not active,',
            '!!  - the current user is not a member of a task-group that has',
            '!!    permissions to use the commands.',
            'hostnaime customer2'],
 'status': 'error'}
[apple2:~]$ 
```


Syslog Messages sent to the server during the run are shown below:

```


2016-12-16T05:14:15-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:16,424", "pathName":"ztp_helper.py", "logRecordCreationTime":"1481952256.424373", "functionName":"download_file", "levelNo":"20", "lineNo":"151", "time":"424", "levelName":"INFO", "message":"Downloading file ncs5500-k9sec-2.2.0.0-r62121I.x86_64.rpm from URL:http://11.11.11.2:9090/packages/ncs5500-k9sec-2.2.0.0-r62121I.x86_64.rpm"}
2016-12-16T05:14:15-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:16,696", "pathName":"ztp_helper.py", "logRecordCreationTime":"1481952256.696958", "functionName":"install_xr_package", "levelNo":"20", "lineNo":"412", "time":"696", "levelName":"INFO", "message":"Package Download complete, starting installation process"}
2016-12-16T05:14:20-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:21,559", "pathName":"ztp_helper.py", "logRecordCreationTime":"1481952261.559930", "functionName":"install_xr_package", "levelNo":"20", "lineNo":"485", "time":"559", "levelName":"INFO", "message":"Installation of ncs5500-k9sec package successsful"}
2016-12-16T05:14:20-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:21,565", "pathName":"ztp_helper.py", "logRecordCreationTime":"1481952261.565898", "functionName":"download_file", "levelNo":"20", "lineNo":"151", "time":"565", "levelName":"INFO", "message":"Downloading file ncs5500-mgbl-3.0.0.0-r62121I.x86_64.rpm from URL:http://11.11.11.2:9090/packages/ncs5500-mgbl-3.0.0.0-r62121I.x86_64.rpm"}
2016-12-16T05:14:21-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:22,984", "pathName":"ztp_helper.py", "logRecordCreationTime":"1481952262.984670", "functionName":"install_xr_package", "levelNo":"20", "lineNo":"412", "time":"984", "levelName":"INFO", "message":"Package Download complete, starting installation process"}
2016-12-16T05:14:26-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:27,819", "pathName":"ztp_helper.py", "logRecordCreationTime":"1481952267.819593", "functionName":"install_xr_package", "levelNo":"20", "lineNo":"485", "time":"819", "levelName":"INFO", "message":"Installation of ncs5500-mgbl package successsful"}
2016-12-16T05:14:37-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:38,903", "pathName":"sample_ztp_script.py", "logRecordCreationTime":"1481952278.903180", "functionName":"<module>", "levelNo":"20", "lineNo":"106", "time":"903", "levelName":"INFO", "message":"Hostname updated"}
2016-12-16T05:14:38-08:00 11.11.11.45 Python: { "loggerName":"ZTPLogger", "asciTime":"2016-12-17 05:24:39,482", "pathName":"sample_ztp_script.py", "logRecordCreationTime":"1481952279.482939", "functionName":"<module>", "levelNo":"20", "lineNo":"107", "time":"482", "levelName":"INFO", "message":"{'status': 'success', 'output': ['hostname customer2']}"}

```











