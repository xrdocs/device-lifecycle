---
published: false
date: '2018-05-15 11:11 -0700'
title: ZTP Workflow Part 2 The DHCP Server
---
## Introduction

The most common way to provide a valid bootfile name (option 67/59) to a device being provisioned is to use the DHCP options. This document lists the relevant option sent by the client and example of usage to provision the device at the DHCP level.
The ZTP client of IOS-XR uses the isc-dhclient version 4.3.0 and specific behavior of this client are described in the [ISC document](https://kb.isc.org/article/AA-00333 ) 
The dhcp client send a list of requested option from the server in option 55 (IPv6 option 6) which of course include option 67 (IPv6 option 56) but also include a default gateway, a domain name, a host name and a DNS server(s) list. These options if provided by the server will be used by the client. Version 6.5.1 add IPv4 option 42 on data ports sourced discover.

### IPv4 example:
Parameter Request List Item: (1) Subnet Mask
Parameter Request List Item: (28) Broadcast Address
Parameter Request List Item: (2) Time Offset
Parameter Request List Item: (3) Router
Parameter Request List Item: (15) Domain Name
Parameter Request List Item: (6) Domain Name Server
Parameter Request List Item: (12) Host Name
Parameter Request List Item: (67) Bootfile name
### IPv6 example:
Requested Option code: DNS recursive name server (23)
Requested Option code: Domain Search List (24)
Requested Option code: Unknown (243)
Requested Option code: Unknown (242)
Requested Option code: Boot File URL (59)
Requested Option code: Unknown (242)
Requested Option code: Fully Qualified Domain Name (39)

## mac-address
mac-address, the mac address is not an option but the source mac address of the DHCP Discover, for a host declaration it must be matched completely but a partial match on the OUI for example can be used in a class. Partial matching on OUI will take more importance when IOS-XR will be available on different HW vendors. 
### Example1:
host ncs-5001-b {
      hardware ethernet c4:72:95:a7:ef:c2;
      if exists user-class and option user-class = "exr-config" {
        filename = "http://192.168.0.22/ncs-b_rp0_ztp.sh";
      } 
      fixed-address 192.168.0.50;
      ddns-hostname "ncs-5001-b";
    }
### Example2:
class “cisco-ncs” {
   match if (substring(hardware,1,3) = c4:72:95 );
}
pool {
     deny members of "cumulus";
     deny members of “Juniper”;
     allow member of “cisco-ncs”;   
     range 192.168.0.30 192.168.0.34;
     option default-url = "http://192.168.0.22/scripts/ncs-generic.sh";
  }

## Vendor-Class-Identifier (IPv4: option 60 and 124. IPv6: option 16)
The PID is part of option vendor-class-identifier (option 60) and the Vendor-Identifying Vendor Class Option (VIVCO) (IPv4 option 124, IPv6 option 16). 

Option 60 is identical in the iPXE and ZTP phases and encodes the following information
### Example:
PXEClient:Arch:00009:UNDI:003010:PID:NCS-5508

The fields can be broken down into:
1.	The type of client: e.g. PXEClient
2.	The architecture of The system (Arch): e.g.: 00009 Identify an EFI system using a x86-64 CPU
3.	The Universal Network Driver Interface (UNDI): e.g.: 003010 (first 3 octets identify the major version and last 3 octets identify the minor version)
4.	The Product Identifier (PID): e.g.: NCS-5508

Option 124: Vendor-Identifying Vendor Class Option (vivco). This option was introduced to have parity with the corresponding DHCPv6 option (16) on which option 124 is modeled. This option encodes the following information:

1. Vendor Enterprise Number: The Cisco IANA Enterprise Number is 9. This is the first 4 bytes within the option and is encoded as an integer 32 in the option.
2.	Serial Number: Option 124/vivco defines a data option field that may contain any vendor specific information (RFC 3925). The first 14 characters identifies the Serial Number: (SN: + <11 character Serial Number>).
3.	Platform PID: The remaining data option field encodes the platform name (NCS-5508 in this example).
Provisioning using the PID is often used for generic provisioning of the device.
Using the PID as identifier can be used to install packages and/or apply SMUS that are common to a specific platform or a set of platforms. More specific configuration are added by the downloaded script itself or through other methods than ZTP. 
### Example1:
```
option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};

class “NCS-5508” {
match if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99) = "NCS-5508")
}
```
### Example2:
```
class "ncs-5001" {
   match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
      if substring (option vendor-class-identifier, 37, 8) = "NCS-5001" {
         filename = "http://192.168.0.22/scripts/ncs5k-mini-ztp.sh";
      }
 }
 ```
### Example3:
```
option dhcp6.vendor-class code 16 = { integer 32, integer 16, string };
if substring (option dhcp6.vendor-class, 43, 99) = "NCS-5508" {
   option dhcp6.bootfile-url = "http://[2001:dba:100::1]/scripts/ncs-5508_ztp.py";
}
```

## Client-Identifier (IPv4: option 61 and 124, IPv6: option 1)
The serial number is sent as part of option 67 (client-identifier) and 124(VIVCO), full matching is required on Serial number.
### Example1:
```
if option dhcp-client-identifier = "FOC1947R143" {
   option bootfile-name = "http://192.168.0.22/scripts/ncs5k-FOC1947R143.sh";
}
```
### Example2:
```
option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};

if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11) = "FGE194714QS") {
   filename="http://192.168.0.22/scripts/exhaustive_ztp_script.py";
}
```
## User-class (IPv4: option 77, IPv6 option 15)
A very generic provisioning mechanism that works across all Cisco platforms supporting ZTP is simply to  use option 77 (user-class), this option is often used to differentiate between the iPXE phase and ZTP phase.
### Example1:
```
class “ZTP” {
   match if exists user-class and option user-class = “exr-config”;
}
```
## Secure Relay (option 82)
Relaying DHCP requests is often necessary in customer deployment in this case the first hop router can implement option 82. This option allow the insertion of special parameters in option 82 to help identify the device.
Option 82 provides information about the network location of a DHCP client, and the DHCP server uses this information to implement IP addresses or other parameters for the client. See RFC 3046, DHCP Relay Agent Information Option, at http://tools.ietf.org/html/rfc3046.
Option 82 comprises the suboptions circuit ID, remote ID, and vendor ID. These suboptions are fields in the packet header:
1. circuit ID: Identifies the circuit (interface or VLAN) on router on which the request was received. The circuit ID may contains the interface name and eventually VLAN name, with the two elements separated by a colon. For example, ge-0/0/10:vlan1.
2. remote ID: Identifies the remote host. 
3. vendor ID: Identifies the vendor of the host
### Example:
```
class "VLAN10" {
        match if binary-to-ascii(10,16,"",substring(option agent.circuit-id,2,2)) = "10";
}
```

