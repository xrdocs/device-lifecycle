---
published: true
date: '2019-03-08 12:38 -0700'
title: ZTP Workflow Part 1 Planning Ahead
author: Patrick Warichet
excerpt: Introduction to IOS-XR ZTP Planning Ahead
tags:
  - iosxr
  - cisco
  - linux
  - ztp
position: hidden
---

## Introduction

ZTP is a powerful framework that allows users to automate device bring up. It was introduced with IOS-XR 6.0 and has since been enhanced with multiple features.
ZTP consists of a set of scripts launched by the process manager of IOS-XR but executed within the Linux environment of the control plane LXC.
Because it is executed within the Linux environment, it has access to all the tools, utilities and libraries available within the IOS-XR Linux distribution.
Access to IOS-XR itself is provided by a series of functions included in a helper library as illustrated below:

![ztp1.png]({{site.baseurl}}/images/ztp1.png)

ZTP was initially developed for the NCS5000 and NCS5500 series but has been extended to support the ASR9K and the NCS540 devices.

The goal of this series of blogs is to go beyond the initial tutorial: [Working with Zero Touch Provisioning](https://xrdocs.io/device-lifecycle/tutorials/2016-08-26-working-with-ztp/) and to help users plan their ZTP deployments.

## Internal Resources

ZTP is executed within the same process name space as IOS-XR and once started, acquires root privileges within that name space. Since ZTP is root on the system, all file manipulations need to be carefully scoped and risk assessments should be made.

If the user script includes the ZTP helper libraries, functions from the library will execute with "root_lr" privileges inside the IOS-XR control plane. It is important that any user script using the ztp helper library is carefully assessed against potential risks.

ZTP is executed within the global-vrf network namespace of IOS-XR and has access to all the interfaces within that network name space as long as those interfaces are up. ZTP can launch processes that will use stream or datagram sockets (no raw socket). In this case the UDP/TCP port needs to be carefully chosen to avoid conflicts with IOS-XR.
Starting with IOS-XR 6.2.25, the ZTP helper library provides a function that can change the current network name space on the fly. This can be usefull if the user script applies a configuration that isolates a set of interfaces in a specific VRF but communication within this newly created VRF is still required by the script.

In summary, ZTP has total control over the device both at the system level and at the control plane configuration level. ZTP goes beyond the initial configuration of the device. It includes capabilities to manipulate the device file system, install applications, deploy containers and more.

## External Resources

ZTP requires two external services:

1. DHCP Server
2. HTTP Server

Details of the DHCP configuration is covered in part 2 of this series and the HTTP server configuration will be covered in part 3.

### DHCP

One of the first actions the ZTP process performs is to acquire an IP address through DHCP (v4/v6). The initial DHCP Discover comes with a set of options that help in the provisioning of the devices. The role of the DHCP server is to provide a valid IP address and a link to a provisioning script or a configuration file that can be downloaded from an HTTP server.
A full review of the DHCP process with packet captures is detailed in the blog post: [IOS-XR ZTP: Learning through Packet Captures](https://xrdocs.io/device-lifecycle/blogs/2017-09-21-ios-xr-ztp-learning-through-packet-captures/).
Details of the DHCP configuration is covered in part 2 of this series.

### HTTP

Once a valid link to a URL is obtained from the DHCP Offer, the ZTP script will fork a CURL process to retrieve the file. We allow all options supported by CURL in this request. Details of the HTTP configuration will be covered in part 3 of this series.

## How to plan for a ZTP deployment
ZTP can perform all operations that an operator does manually on the system, but in order to automate these operations, there are a few criteria that should be considered. Here is a list of important ones:

### Generic and Specific
Since ZTP can potentially be used on a large set of different type of devices it is important to separate identical tasks that can be applied on these devices from more specific ones.

An initial ZTP planning should start by reviewing the typical workflow of a new device deployment and seperate generic tasks from the one that are unique to the device. With this approach, the operator will keep the set of scripts to a minimum which in turn reduces the effort to maintain these scripts.

The method used to identify devices that can be configured similarly is up to the operator but is usually based on the role of the device or the type of device.

#### Generic
For a specific role, this can include:

* Device Domain Name
* Security Access Lists
* QoS Policies
* Routing Policies
* Filter Lists
* Third Party Packages
* Containers
* etc.

For a specific type of device, this generally includes items like:

* IOS-XR Packages to be installed
* Port Queue Configuration
* RPs Synchronization (if multiple RP system)
* etc.

#### Specific
Specific configuration items should be kept at a minimum and should be called from a generic configuration script.
Specific configuration generally include:

* Device Name
* OSPF area
* BGP AS
* BGP Neighbors List
* etc.

### Static and Dynamic
Static provisioning refers to the configuration of the device based on its attributes alone. The most commonly used attribute is the device serial number. Other attributes can include the PID of the device, the mac address of the management interface, etc.
Static provisioning is the easiest method, but may be harder to maintain if applied to a large set of devices.

Dynamic provisioning refers to the configuration of the device based on its discovered position inside the network. This method will require more processing on the back end and generally includes a database where the relation between devices triggers a configuration snippet to be applied.

With Dynamic provisioning the initial ZTP script will perform several operations on the system (enable netconf, unshut interfaces, configure LLDP, etc.), collect data (link status, LLDP neighbors, etc.), encode this data in a specific format (XML, JSON, etc.) and push this data to the provisioning server, the provisioing server will use this data as keys to create configuration snippets and push these configuration excerpts to the device using netconf for example.  
