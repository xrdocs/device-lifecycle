---
published: true
date: '2019-03-14 11:11 -0700'
title: ZTP Workflow Part 2 The DHCP Server
author: Patrick Warichet
tags:
  - iosxr
  - cisco
  - linux
  - ZTP
position: hidden
---
# Introduction

ZTP relies on DHCP to obtain the URL of a bootstrap file, In this blog we will go over the list of relevant options sent by the client and DHCP server configuration example that can be used to efficiently provision IOS-XR devices via ZTP.
The ZTP client of IOS-XR uses the isc-dhclient version 4.3.0 and specific behavior of this client are described in the [ISC document](https://kb.isc.org/article/AA-00333) 
ZTP configures the DHCP client with distinct options and parameters during its initialization sequence.

The ZTP DHCP client sends a list of requested options from the server in option 55 (IPv6 option 6). Every DHCP server must include these options in its reply if they are available.
The most important option in the list is "boot-filename" (option 67) or DHCPv6 option "OPT_BOOTFILE_UR" (option 59). These options if provided by the server will be used by the client to provision the system.

IOS-XR version 6.5.1 adds option 43 "vendor-specific" (IPv6 option 17) to the list when ZTP is activated on data ports.

## List of options requested  

**IPv4:**

- Option 1: Subnet Mask
- Option 2: Time Offset (not used)
- Option 3: Routers
- Option 6: Domain Name Server
- Option 12: Host Name
- Option 15: Domain Name
- Option 28: Broadcast Address
- Option 43: Vendor Specific (Data Port only) 
- Option 67: Bootfile Name

**IPv6:**

- Option 17: Vendor-specific Information (Data Port only)
- Option 23: DNS recursive name server
- Option 24: Domain Search List
- Option 39: Fully Qualified Domain Name
- Option 59: Boot File URL
- Option 242: Next Hop (without RT Prefix)

## List of option advertised

**IPv4:**

- Option 61: Client Identifier
- Option 77: User Class Information
- Option 124: Vendor Identifying Vendor Class (VIVCO)

**IPv6:**

- Option 1: Client Identifier
- Option 15: User Class
- Option 16: Vendor Class


## Mac Address
The mac address provides unambiguous identification of a particular interface on the router and has been used for years in network operations. In the particular case of an IOS-XR router, the mac-address may not the best choice to identify devices, especially when the device is provisionned from the data port in a modular system. For this reason, we do not recommend using the mac address to identify the host but to use the serial number instead.

In IPv4, the mac-address is not an option but is included in the header of the DHCP Discover/Request.

Contrary to IPv4, IPv6 does not include the source mac address in the header but rely on the DHCP Unique Identifier (DUID) (option 1). ZTP implements DUID type 2 (Vendor-assigned unique ID based on Enterprise Number) or DUID-EN with the identifier containing serial number of the device encoded in hex.

> **Note:** Provisioning based on mac address is not available in IPv6 as ZTP does not send DUID type 1 (DUID-LLT) or type 3 (DUID-LL) witch is usually mapped to the mac-address.

If you decide to use the mac address to identify the device in the DHCP server you can use a host declaration. within a host declaration, the mac address must be matched completely but a partial match on the OUI for example can be used in a class.

Partial matching on OUI will take more importance when IOS-XR will be available on different HW vendors.

**Example1:**
Using the mac address in a host declaration

```
host ncs-5001-b {
      hardware ethernet c4:72:95:a7:ef:c2;
      if exists user-class and option user-class = "iPXE" {
        filename = "http://192.168.0.22/ncs5000/6.5.1/ncs5k-mini-x-6.5.1.iso";
      elseif exists user-class and option user-class = "exr-config" {
        option bootfile-name = "http://192.168.0.22/scripts/ncs-b_rp0_ztp.sh";
      } 
      fixed-address 192.168.0.50;
      ddns-hostname "ncs-5001-b";
    }
```

In this example, the user class (option 77) is used to differentiate between the iPXE (user-class = "iPXE")and ZTP (user-class = "exr-config") phases of the provisioning process see the section on option 77 for more details.

> **Note:** in this example the DHCP server is configured to support dynamic host name, this feature is very useful to immediatly provide access to the device by name. For more information you can refer to the following wiki [DHCP and DDNS](https://wiki.debian.org/DDNS)

**Example2:**

Using the Organizational Unique Identifier (OUI) in a class

```
class "cisco" {
   match if (substring(hardware,1,3) = c4:72:95 );
}
pool {
     deny members of "linux-hosts";
     allow member of "cisco";   
     range 192.168.0.30 192.168.0.34;
     if exists user-class and option user-class = "exr-config" {
       option default-url = "http://192.168.0.22/scripts/ztp-generic.sh";
     }
  }
```

## Vendor Specific
(ipv4 option 43, IPv6 option 17)

The Vendor Specific option was introduced with the release of IOS-XR 6.5.1, and it is only requested when ZTP is initiated automatically on the Data Port.


> **Note:** A manual invokation of ZTP using the ztp initiate command will not request this option.

The goal of this option is to adequate the DHCP server with the device being provisioned. This option is structured as follow:

* code 1 is a string representing the client identification and must be equal to "exr-config" 
* code 2 is an unsigned integer 8 represeting the type of authorization the device should accept
  * 0 = no authentication
  * 1 = md5 checksum
* code 3 is a string representing the md5 checksum of the device's serial number if code 2 = 1

**Example 1:**
option 43 definition for IPv4

```
option space VendorInfo;
option VendorInfo.clientId code 1 = string;
option VendorInfo.authCode code 2 = unsigned integer 8;
option VendorInfo.md5sum code 3 = string;
option vendor-specific code 43 = encapsulate VendorInfo;
```

**Example2:**
option 17 definition for IPv6
```
option space CISCO_EXR_CONFIG code width 2 length width 2;
option CISCO-EXR_CONFIG.client-identifier code 1 = string;
option CISCO-EXR_CONFIG.authCode code 2 = unsigned integer 8;
option CISCO-EXR_CONFIG.md5sum code 3 = string;
option vsio.CISCO-EXR_CONFIG code 17 = encapsulate CISCO-E XR_CONFIG;
```

If the intend is to provision the device through the data port, the DHCP server must include the definition and the values of option 43 or option 17. The user can choose to use md5sum for a more strict verification or no authentication for a less strict verification.
To generate an md5 checksum of the device serial number, you can use the following command on a Linux system and paste the result in the dhcp configuration file.

```
echo -n "FGE18410JSL" | md5sum | awk '{print $1}’
```

**Example 3:**

Host declaration for ZTP IPv4 on data port without md5 checksum
This example use the definition described in example 1

```
host ncs-5501-1 {
   option dhcp-client-identifier "FOC2050R0BV";
   vendor-option-space VendorInfo;
   option VendorInfo.clientId "exr-config";
   option VendorInfo.authCode 0;
   if exists user-class and option user-class = "iPXE" {
      filename "http://10.195.122.236/ISO/ncs5500-mini-x-6.5.1.iso";
   } elsif exists user-class and option user-class = "exr-config" {
      filename "http://10.195.122.236/scripts/ztp_script.sh";
   }
}
```

**Example 4:**

Host declaration for ZTP IPv4 on data port including MD5 checksum verification.
This example use the definition described in example 1

```
host ncs-5501-1 {
   option dhcp-client-identifier "FOC2050R0BV";
   vendor-option-space VendorInfo;
   option VendorInfo.clientId "exr-config";
   option VendorInfo.authCode 1;
   option VendorInfo.md5sum "aedf5c457c36390c664f5942ac1ae3829";
   if exists user-class and option user-class = "iPXE" {
      filename "http://10.195.122.236/ISO/ncs5500-mini-x-6.5.1.iso";
   } elsif exists user-class and option user-class = "exr-config" {
      filename "http://10.195.122.236/scripts/ztp_script.sh";
   }
}
```

**Example 5:**

Host declaration for ZTP IPv6 on data port without authentication
This example use the definition described in example 2

```
host ncs-5501-1 {
   host-identifier option dhcp6.client-id 00:02:00:00:00:09:46:4f:43:32:30:32:34:52:31:55:58:00; 
   option CISCO-EXR-CONFIG.client-identifier "exr-config";    
   option CISCO-EXR-CONFIG.authCode 0;
   if exists dhcp6.user-class and substring(option dhcp6.user-class, 2, 4) = "iPXE" {  
	option dhcp6.bootfile-url  "http://[6::1]/R1/ncs5500-mini-x-6.5.1.iso";
   } else if exists dhcp6.user-class and substring(option dhcp6.user-class, 0, 10) = "exr-config" { 
	option dhcp6.bootfile-url "http://[6::1]/R1/ztp_script.sh";
   }
}
```

**Example 6:**
Host declaration for ZTP IPv6 on data port with md5 checksum authentication
This example use the definition described in example 2

```
host ncs-5501-1 {
   host-identifier option dhcp6.client-id 00:02:00:00:00:09:46:4f:43:32:30:32:34:52:31:55:58:00; 
   option CISCO-EXR-CONFIG.client-identifier "exr-config";    
   option CISCO-EXR-CONFIG.authCode 1;
   option CISCO-EXR-CONFIG.md5sum "aedf5c457c36390c664f5942ac1ae3829";
   if exists dhcp6.user-class and substring(option dhcp6.user-class, 2, 4) = "iPXE" {  
	option dhcp6.bootfile-url  "http://[6::1]/R1/ncs5500-mini-x-6.5.1.iso";
   } else if exists dhcp6.user-class and substring(option dhcp6.user-class, 0, 10) = "exr-config" { 
	option dhcp6.bootfile-url "http://[6::1]/R1/ztp_script.sh";
   }
}
```

## Class-Identifier
(IPv4: option 60)

Option 60 is identical in the iPXE and ZTP phases and encodes the following information:

1.	The type of client: e.g. PXEClient
2.	The architecture of The system (Arch): e.g.: 00009 Identify an EFI system using a x86-64 CPU
3.	The Universal Network Driver Interface (UNDI): e.g.: 003010 (first 3 octets identify the major version and last 3 octets identify the minor version)
4.	The Product Identifier (PID): e.g.: NCS-5508

**Example**

```
PXEClient:Arch:00009:UNDI:003010:PID:NCS-5508
```

The important information that can be retrieved from option 60 is the Product Identifier (PID). 
Provisioning using the PID is often used for generic provisioning of the device.
Using the PID as identifier can be used to install packages and/or apply SMUS that are common to a specific platform or a set of platforms. More specific configuration are added by the downloaded script itself or through other methods than ZTP.

The PID is also part of the Vendor-Identifying Vendor Class Option (VIVCO) (IPv4 option 124, IPv6 option 16).

**Example1:**

```
class "NCS-5501" {
  match if exists vendor-class-identifier and substring(option vendor-class-identifier,0,9) = "PXEClient" and substring(option vendor-class-identifier,37,8) = "NCS-5501";
  if exists user-class and option user-class = "iPXE" {
    filename "http:///192.168.0.22/ncs5500/6.5.1/ncs5500-mini-x-6.5.1.iso";
  } else {
    filename "http://192.168.0.22/scripts/ztp-generic.sh";
  }
}
```

## Client-Identifier
(IPv4: option 61, IPv6: option 1)

Option 61 represent the serial number of the device, the serial number is also sent as part of option 124 (VIVCO).
The serial number advertised by the device is the one of the chassis and is written on the shipping box which facilitate the pre-provisioning of the box. Since it is the chassis serial number, it is persistent on modular system where the RP can be replaced. It is also unique in a dual RP system.

For these reasons, the serial number is the recommended approach to identify the device, it is valid in both IPv4 and IPv6 but since there is no hierachy in the serial number, it need to be  matched in its entirety.

**Example1:**
The following example can be part of the pool or subnet declaration.

```
if option dhcp-client-identifier = "FOC1947R143" {
   option bootfile-name = "http://192.168.0.22/scripts/ncs5k-FOC1947R143.sh";
}
```

**Example2:**
The following example uses a host declaration based on the device's serial number

```
host ncs-5001-1 {
   option dhcp-client-identifier "FOC1947R143";
   if exists user-class and option user-class = "iPXE" {
      filename = "http://192.168.0.22/ncs5000/6.3.2/ncs5k-mini-x-6.3.2.iso";
   }
   elsif exists user-class and option user-class = "exr-config" {
      filename = "http://192.168.0.22/scripts/ztp.script.xrapply";
   }
       fixed-address 192.168.10.56;
       ddns-hostname "ncs-5001-1";
       option routers 192.168.10.1;
}
```

**Example3:**

In this example the device will also use DHCP to configure the management interface IPv4 address inside XR if the management interface is configured like this:

```
interface MgmtEth0/RP0/CPU0/0
 ipv4 address dhcp
```

```
class "ncs-5001-1" {
  match if (option dhcp-client-identifier = "MgmtEth0_RP0_CPU0_0.c4:72:95:a7:ef:c0") or (option dhcp-client-identifier = "FOC1947R143");
}

pool {
  allow members of "ncs-5001-1";
  range 172.30.0.70 172.30.0.71;
  if exists user-class and option user-class = "iPXE" {
    filename = "http://192.168.00.22/ncs5000/6.5.1/ncs5k-mini-x-6.5.1.iso";
  }
  elsif exists user-class and option user-class = "exr-config" {
    filename = "http://192.167.0.22/scripts/ztp.script.xrapply";
  }
  ddns-hostname "ncs-5001-1";
}
```

IOS-XR uses a different DHCP client for the management interface with a different default format for option 61. This format can be changed using the following configuration command:

```
RP/0/RP0/CPU0:ios(config-if)#ipv4 address dhcp-client-options option 61 ?
    ascii       Option 61 in ascii
    sn-chassis  sn chassis number
```

## Bootfile name
(IPv4: option 67, IPv6 option 59)

Option 67 is a standard DHCPv4 option of type string and identified as the "bootfile-name", it is a free form string that should include the full URL of the file to be retrieved, this option is provided by the server to identify the file to be used to start the auto configuration process. Option 67 is supported starting with IOS-XR 6.2.25 previous release will only support the bootp option "filename" which is deprecated.

The URL should point to either a configuration file or a script, scripts can be written in either python (version 2.7.3) or bash (version 4.3.30).

ZTP supports both HTTP and HTTPS with self-signed certificate only. If you plan to use self-signed certificate with HTTPS, you will have to pass the "-k" switch at the beginning of the URL string. The certificate provided by the server must be valid (i.e. not expired) and the ip address inside the certificate must match the ip address of the HTTPS server.

ZTP uses the curl agent to download the file provided by the DHCP server and will performs some checks based on the header of the first HTTP segment. If the file is not a text file or the file is larger than 100 MB ZTP will erase the file and restart its execution.

After downloading the file, ZTP will analyze the first line of the text file received, if the first line of the file starts with "!! IOS XR", it will consider it as a configuration file and pass it to the command line interpreter for syntax verification and commit it. If the first line starts with "#!/bin/bash" or "#!/bin/sh" or "#!/usr/bin/env python" ZTP will assume this is a script and start the execution within the Linux environamnet.

**Example1:**
Passing an HTTP URL for a shell configuration script

```
option bootfile-name = "http://172.30.0.22/scripts/ncs-5001-rp0_ztp.sh"
```

**Example2:**
Passing an HTTPS URL for a python configuration script, notice the "-k" switch passed to the curl agent.

```
option bootfile-name = "-k https://172.30.0.25/scripts/ncs-5501-rp0_nso.py"
```

**Example3:**
Passing an HTTP URL for an HTTP server requiring clear-text authentication

```
option bootfile-name = "http://username:password@192.168.1.22:8080/ncs-5508-core.conf"
```


## User-class
(IPv4: option 77, IPv6 option 15)

The user-class option is used to indicate the phase of IOS-XR provisioning process, it can only carry the following 2 values:

1. **"IPXE"** : When the device is in iPXE mode and requires an ISO image
2. **"exr-config"** : When the device is in ZTP mode and requires a configuration or a script

A very generic provisioning mechanism that works across all Cisco platforms supporting ZTP is simply to use option 77 (user-class) in a DHCP class.

**Example1:**

```
class “ZTP” {
   match if exists user-class and option user-class = “exr-config”;
}
```

## Vendor-Class
(IPv4: option 124, IPv6 option 16)

The full name of option 124 is "Vendor-Identifying Vendor Class Option" often refer to as VIVCO. This option was introduced to have parity with the corresponding DHCPv6 option 16 (Vendor Class) on which option 124 is modeled. This option encodes the following information:

1. Vendor Enterprise Number:
The Cisco IANA Enterprise Number is 9. This is the first 4 bytes within the option and is encoded as an integer 32 in the option.
2. Serial Number:
Option 124/vivco defines a data option field that may contain any vendor specific information (RFC 3925). The first 14 characters identifies the Serial Number: (SN: + <11 character Serial Number>).
3. Platform PID:
The remaining data option field encodes the platform name (NCS-5508 for example).


**Example1:**

IPv4 option 124 requires to create an option space that define the format of the option fields.
This example defines a class that matches all NCS-5508 devices

```
# Option 124 (VIVCO)
option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};

class “NCS-5508” {
   match if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99) = "NCS-5508")
}
```
**Example2:**

Same as example 1 but using IPv6.

```
# Option 16 (VIVCO) 
option dhcp6.vendor-class code 16 = { integer 32, integer 16, string };

if substring (option dhcp6.vendor-class, 43, 99) = "NCS-5508" {
   option dhcp6.bootfile-url = "http://[2001:dba:100::1]/scripts/ncs-5508_ztp.py";
}
```

**Example3:**

This example will match a device based on its serial number.

```
option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};

if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11) = "FGE194714QS") {
   filename = "http://192.168.0.22/scripts/exhaustive_ztp_script.py";
}
```


## Relay and Gateway
If the device is behind a router, the router must relay the DHCP Discover broadcast to either the unicast address of the DHCP server or a targeted broadcast for the subnet of the DHCP server. The second option offers more flexibility especially if multiple DHCP servers are present in Active-Standby scenario.

The relay agent must have a route to the subnet of the DHCP server and the DHCP server must have a route to the subnet of the DHCP client device.

In this case the DHCP server must also provide option 3 (routers) in its offer for IPv4.

In IPv6 the traditional way for a client to obtain the address of a gateway is through Router Advertisement (RA) send by routers present on the client subnet. Unfortunatly ZTP cannot process RAs at this point in time. In place of relying on RA, ZTP requests DHCPv6 option 242 (NEXT_HOP) from the DHCP server, ZTP implement the simple version of option 242 (without RT_LINK fields) and the server should be configured to only provide the IPv6 address of the gateway.

```
# Option 242 simplified mode (NEXT_HOP only, without RTPREFIX)
option dhcp6.next-hop code 242 = ip6-address;

shared-network FD-31-0 {
  subnet6 fd:31::/64 {
    # Range for clients
    range6 fd:31::1024 fd:31::1124;
    # Range for clients requesting a temporary address
    range6 fd:31::/64 temporary;
    # Additional options
    option dhcp6.name-servers fd:30::172:30:0:25;
    option dhcp6.domain-search "cisco.lab";
    option dhcp6.next-hop fd:31::1;
 }
 host ncs-5001-1 {
  host-identifier option dhcp6.client-id 00:02:00:00:00:09:46:4f:43:31:39:34:37:52:31:34:33:00;
  if exists dhcp6.user-class and substring(option dhcp6.user-class, 2, 4) = "iPXE" {
    option dhcp6.bootfile-url = "http://[fd:30::172:30:0:22]/ncs5000/6.5.1/ncs5k-mini-x-6.5.1.iso";
  } else if exists dhcp6.user-class and substring(option dhcp6.user-class, 0, 10) = "exr-config" {
    option dhcp6.bootfile-url = "http://[fd:30::172:30:0:22]/scripts/ztp.script.xrapply";
  }
    fixed-address6 fd:31::172:31:0:70;
  }
 }
```

## Secure Relay
(IPv4 option 82)

Relaying DHCP requests is often necessary in customer deployment in this case the first hop router can implement option 82. This option allow the insertion of special parameters in option 82 to help identify the device.
Option 82 provides information about the network location of a DHCP client, and the DHCP server uses this information to provides IP addresses or other parameters for the client. See RFC 3046, DHCP Relay Agent Information Option, at http://tools.ietf.org/html/rfc3046.
Option 82 comprises the suboptions circuit ID, remote ID, and vendor ID. These suboptions are fields in the packet header:
1. circuit ID: Identifies the circuit (interface or VLAN) on router on which the request was received. The circuit ID may contains the interface name and eventually VLAN name, with the two elements separated by a colon. For example, ge-0/0/10:vlan1.
2. remote ID: Identifies the remote host. 
3. vendor ID: Identifies the vendor of the host

**Example:**

```
class "VLAN10" {
        match if binary-to-ascii(10,16,"",substring(option agent.circuit-id,2,2)) = "10";
}
```
