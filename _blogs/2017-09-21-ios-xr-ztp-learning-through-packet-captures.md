---
published: true
date: '2017-09-21 13:19 -0700'
title: 'IOS-XR ZTP: Learning through Packet Captures'
author: Akshat Sharma
excerpt: >-
  Packet Captures are a great way to understand how the router running IOS-XR
  behaves with the outside world. In this blog,  we look at Zero Touch
  provisioning workflows and the corresponding packets exchanged with the DHCP
  server
tags:
  - iosxr
  - cisco
  - linux
  - ZTP
  - DHCPv6
  - dhcp
  - dhcpv4
  - option 124
  - vivco
  - pcap
  - cloudshark
  - wireshark
position: top
---


{% include toc %}

# Introduction

Zero Touch Provisioning is quite often considered the cornerstone of web scale deployment practices. It brings forth a mindset that is quite emblematic of the way Large Scale (Web) Service providers provision their environment:   

>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*"The network device should never have to be configured manually through its console - from the instant that it is powered on right up until the services are brought up and [telemetry](https://xrdocs.github.io/telemetry/) data starts flowing."*
    
There are dozens of techniques that vendors have brought forward over the years to enable users to push a configuration down to their devices often using on-premise DHCP servers and encoding the communication with the device within DHCP options. Over the years as devices became natively scriptable, a script of the user's choice may be downloaded to automate provisioning of the entire system (NOS, agents, scripts etc.) on the whole and not just the configuration of the Network OS.    

IOS-XR is no exception and as Patrick's excellent tutorial illustrates: 
><https://xrdocs.io/device-lifecycle/tutorials/2016-08-26-working-with-ztp/>

a lot of headway has been made in enabling IOS-XR to be completely scriptable as soon as it is powered on.

To understand the variety of options available to classify the router when it first identifies itself to the DHCP server and to supply the required script, let's look at packet captures in the following scenarios:

*   **DHCPv4 using BOOTP filename** to supply script/config location
*   **DHCPv4 using Option 67 (bootfile-name)** to supply script/config location.
*   **DHCPv6 using Option 59 (OPT_BOOTFILE_URL)** to supply script/config location.




>The packet captures of complete DHCP transactions  along with a sample isc-dhcp server config for each of the above cases has been made available on Github here:
>
><https://github.com/ios-xr/ztp-pcap>



# DHCPv4 using BOOTP filename


For this purpose we use the the DHCPv4 server configuration located [here](https://github.com/ios-xr/ztp-pcap/blob/master/6225/isc-dhcp-server-config/dhcpdv4-filename.conf).

The config is reproduced here for the reader's convenience:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
<span style="background-color: #98FF98">option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};</span>

######### Network 11.11.11.0/24 ################
shared-network 11-11-11-0 {

####### Pools ##############
  subnet 11.11.11.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
	option broadcast-address 11.11.11.255;
	option routers 11.11.11.2;
	option domain-name-servers 11.11.11.2;
	option domain-name "cisco.local";
	# DDNS statements
  	ddns-domainname "cisco.local.";
	# use this domain name to update A RR (forward map)
  	ddns-rev-domainname "in-addr.arpa.";
  	# use this domain name to update PTR RR (reverse map)

    }

######## Matching Classes ##########

  class "ncs5508" {
      <span style="background-color: #FDD7E4"> match if (substring(option dhcp-client-identifier,0,11) = "FGE194714QS");</span>
  }


  pool {
      allow members of "ncs5508";
      range 11.11.11.47 11.11.11.50;
      next-server 11.11.11.2;

      if exists user-class and option user-class = "iPXE" {
         filename="http://11.11.11.2:9090/ncs5500-mini-x-6.2.25.10I.iso";
      }

  
      if exists user-class and option user-class = "exr-config" {
       <span style="background-color: #98FF98">if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)="FGE194714QS")</span> 
        {
          <span style="background-color: #98FF98">if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="NCS-5508")</span>
          {           
            <mark>filename="http://11.11.11.2:9090/scripts/exhaustive_ztp_script.py";</mark>
          }
        }
      }

      ddns-hostname "ncs-5508-local";
      option routers 11.11.11.2;
  }
}
</code>
</pre>
</div>



## Packet Captures


All the relevant packet captures for this scenario have been shared on cloudshark: <https://www.cloudshark.org/captures/2b821c09ed7b> and are embedded below.

>You can also find the corresponding .pcap file in the github repo here: <https://github.com/ios-xr/ztp-pcap/blob/master/6225/pcap/dhcpv4-filename.pcap>



### DHCP Discover (Router to DHCP Server)

Play around with the embedded cloudshark decode of the DHCP discover packet below:

<iframe src="https://www.cloudshark.org/captures/2b821c09ed7b/decode?frame=1&window=yes"
  style='width:100%; height: 200px;border: 5px solid #ccc; border-radius: 10px;'></iframe>
<div class="notice--info" style="margin:0em 0 !important;">
<div class="text-center">
    Interact above or <a href="https://www.cloudshark.org/captures/2b821c09ed7b">View on Cloudshark</a>
  </div>
</div>
&nbsp;  
&nbsp;  

The DHCP discover packet sent by the router encodes the following information:

*  **dhcp-client-identifier (option 61)**: This encodes the Serial Number of the device as a string.  On the DHCP server this option can be used to identify the device during [iPXE](https://xrdocs.github.io/software-management/tutorials/2016-07-27-ipxe-deep-dive/) as well as during the ZTP phase based solely on the Serial Number of the device identified in the purchase order. This unique identification allows complete automation of the device bringup (image and config/script download without ever logging into the device). In the DHCP server configuration above, this match is identified in red.


*  **vendor-class-identifier (option 60)** : This option encodes the following information in this example:  
       
   `PXEClient:Arch:00009:UNDI:003010:PID:NCS-5508`  
     
   and is identical in the iPXE and ZTP phases. This can be broken down into:
   
    *  **The type of client**: e.g. PXEClient 
    *  **The architecture of The system (Arch)**: e.g.: 00009 Identify an EFI system using a x86-64 CPU
    *  **The Universal Network Driver Interface (UNDI)**: e.g.: 003010 (first 3 octets identify the major version and last 3 octets identify the minor version)
    *  **The Product Identifier (PID)**: e.g.: NCS-5508
    
*  **Vendor-Identifying Vendor Class Option (vivco/option 124)**: This option can be used to enforce more granular classification of the network device and the isc-dhcp config that processes this information is marked in lime green (<span style="background-color: #98FF98">   </span>) in the dhcp server configuration above. This option was introduced to have parity with the corresponding DHCPv6 option (16) on which option 124 is modeled.    
    This option encodes the following information:

    *  **Vendor Enterprise Number**:  [The Cisco IANA Enterprise Number is 9](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers). This is the first 4 bytes within the option and is encoded as an integer 32 in the option.
    *  **Serial Number**: Option 124/vivco defines a data option field that may contain any vendor specific information ([RFC 3925](https://tools.ietf.org/html/rfc3925)). The first 14 characters identifies the Serial Number: (SN: + <11 character Serial Number>). 
    *  **Platform PID**: The remaining data option field encodes the platform name (NCS-5508 in this example).
    
*  **User Class Information (Option 77)**:  This option is used by the client (router) to identify the current stage of operation. There are two provisioning stages :
    * **IPXE**:  This is the image download phase and is triggered when the device is brought up without an image or is forced into the iPXE mode in the BIOS. When the network device is performing iPXE, the user-class information encodes the string:  `iPXE`
    * **ZTP**:  This is the stage during which the network device expects to download the configuration/provisioning script that it should apply/execute. During this phase, the user-class information encodes the string: `exr-config`
    
Don't be alarmed by the "malformed option" indication in the packet capture embedded above for option 77. This is a known issue with wireshark due to the variation in the interpretation of option 77 ( see <https://ask.wireshark.org/questions/35332/dhcp-option-77-malformed-option/55006>). 
{: .notice--warning}


### DHCP Offer (DHCP Server to Router)

The cloudshark decode of the DHCP offer message in response to the request described above is shown below:  


<iframe src="https://www.cloudshark.org/captures/2b821c09ed7b/decode?frame=2&window=yes"
 style='width:100%; height: 200px;border: 5px solid #ccc; border-radius: 10px;'></iframe>
<div class="notice--info" style="margin:0em 0 !important;">
<div class="text-center">
    Interact above or <a href="https://www.cloudshark.org/captures/2b821c09ed7b">View on Cloudshark</a>
  </div>
</div>
&nbsp;  
&nbsp;  
While there are multiple options that the Server responds with, there are certain options/fields in particular that the IOS-XR ZTP infrastructure utilizes to determine the location of the script/config to download and the routing required to reach the web server. These are:

*  **BOOTP Filename**: While some DHCP servers might consider this obsolete (for e.g. KEA DHCP Server), there are a few legacy DHCP servers capable of setting and returning the BOOTP filename (like ISC-DHCP 4.x). When this is done (as shown by the yellow (<mark>&nbsp;&nbsp;</mark>) highlight), the filename gets set in the BOOTP flags ( expand the Bootp flags in the DHCP offer capture above). **IOS-XR will accept this and download the target file based on it. IOS-XR will parse the BOOTP filename to identify the server IPv4 address and filename separately.**

*  **Router (Option 3)**: As explained above, IOS-XR uses the filename to parse the server IPv4 address and combines it with the Router IP (option 3) as the gateway to install a static route towards the server IP during the ZTP process. 


The remaining two captures in <https://www.cloudshark.org/captures/2b821c09ed7b>  identify the DHCP request/reply exchange to wrap up the complete exchange between the router and the DHCP server.  




# DHCPv4 using Option 67 bootfile-name (IOS-XR Release 6.2.25+)

Post IOS-XR release version 6.2.25, IOS-XR ZTP also supports explicitly setting the bootfile-name (option 67) in place of the BOOTP filename explained above.

For this purpose we use the the DHCPv4 server configuration located [here](https://github.com/ios-xr/ztp-pcap/blob/master/6225/isc-dhcp-server-config/dhcpdv4-bootfile-name.conf).

The config is reproduced here for the reader's convenience:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
<span style="background-color: #98FF98">option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};</span>

######### Network 11.11.11.0/24 ################
shared-network 11-11-11-0 {

####### Pools ##############
  subnet 11.11.11.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
	option broadcast-address 11.11.11.255;
	option routers 11.11.11.2;
	option domain-name-servers 11.11.11.2;
	option domain-name "cisco.local";
	# DDNS statements
  	ddns-domainname "cisco.local.";
	# use this domain name to update A RR (forward map)
  	ddns-rev-domainname "in-addr.arpa.";
  	# use this domain name to update PTR RR (reverse map)

    }

######## Matching Classes ##########

  class "ncs5508" {
      <span style="background-color: #FDD7E4"> match if (substring(option dhcp-client-identifier,0,11) = "FGE194714QS");</span>
  }


  pool {
      allow members of "ncs5508";
      range 11.11.11.47 11.11.11.50;
      next-server 11.11.11.2;

      if exists user-class and option user-class = "iPXE" {
         filename="http://11.11.11.2:9090/ncs5500-mini-x-6.2.25.10I.iso";
      }

  
      if exists user-class and option user-class = "exr-config" {
       <span style="background-color: #98FF98">if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)="FGE194714QS")</span> 
        {
          <span style="background-color: #98FF98">if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="NCS-5508")</span>
          {           
            <mark>option bootfile-name "http://11.11.11.2:9090/scripts/exhaustive_ztp_script.py";</mark>
          }
        }
      }

      ddns-hostname "ncs-5508-local";
      option routers 11.11.11.2;
  }
}
</code>
</pre>
</div>



## Packet Captures


All the relevant packet captures for this scenario have been shared on cloudshark: <https://www.cloudshark.org/captures/bba324c2b261> and are embedded below.

>You can also find the corresponding .pcap file in the github repo here: <https://github.com/ios-xr/ztp-pcap/blob/master/6225/pcap/dhcpv4-bootfilename.pcap>



### DHCP Discover (Router to DHCP Server)

Take a look at embedded cloudshark decode of the DHCP discover packet.

<iframe src="https://www.cloudshark.org/captures/bba324c2b261/decode?frame=1&window=yes"
 style='width:100%; height: 200px;border: 5px solid #ccc; border-radius: 10px;'></iframe>
<div class="notice--info" style="margin:0em 0 !important;">
<div class="text-center">
    Interact above or <a href="https://www.cloudshark.org/captures/bba324c2b261">View on Cloudshark</a>
  </div>
</div>
&nbsp;  
&nbsp;  

 
The DHCP discover packet sent by the router encodes the following information:

*  **dhcp-client-identifier (option 61)**: This encodes the Serial Number of the device as a string.  On the DHCP server this option can be used to identify the device during [iPXE](https://xrdocs.github.io/software-management/tutorials/2016-07-27-ipxe-deep-dive/) as well as during the ZTP phase based solely on the Serial Number of the device identified in the purchase order. This unique identification allows complete automation of the device bringup (image and config/script download without ever logging into the device). In the DHCP server configuration above, this match is identified in red.


*  **vendor-class-identifier (option 60)** : This option encodes the following information in this example:  
       
   `PXEClient:Arch:00009:UNDI:003010:PID:NCS-5508`  
     
   and is identical in the iPXE and ZTP phases. This can be broken down into:
   
    *  **The type of client**: e.g. PXEClient 
    *  **The architecture of The system (Arch)**: e.g.: 00009 Identify an EFI system using a x86-64 CPU
    *  **The Universal Network Driver Interface (UNDI)**: e.g.: 003010 (first 3 octets identify the major version and last 3 octets identify the minor version)
    *  **The Product Identifier (PID)**: e.g.: NCS-5508  
         
         
    
*  **Vendor-Identifying Vendor Class Option (vivco/option 124)**: This option can be used to enforce more granular classification of the network device and the isc-dhcp config that processes this information is marked in green (<span style="background-color: #00FF00">   </span>) in the dhcp server configuration above. This option was introduced to have parity with the corresponding DHCPv6 option (16) on which option 124 is modeled.    
    This option encodes the following information:

    *  <b>Vendor Enterprise Number</b>:  [[The Cisco IANA Enterprise Number is 9](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers). This is the first 4 bytes within the option and is encoded as an integer 32 in the option.
    *  **Serial Number**: Option 124/vivco defines a data option field that may contain any vendor specific information ([RFC 3925](https://tools.ietf.org/html/rfc3925)). The first 14 characters identifies the Serial Number: (SN: + <11 character Serial Number>). 
    *  **Platform PID**: The remaining data option field encodes the platform name (NCS-5508 in this example).  
       
       
    
*  **User Class Information (Option 77)**:  This option is used by the client (router) to identify the current stage of operation. There are two provisioning stages :
    * **IPXE**:  This is the image download phase and is triggered when the device is brought up without an image or is forced into the iPXE mode in the BIOS. When the network device is performing iPXE, the user-class information encodes the string:  `iPXE`
    * **ZTP**:  This is the stage during which the network device expects to download the configuration/provisioning script that it should apply/execute. During this phase, the user-class information encodes the string: `exr-config`
    
Don't be alarmed by the "malformed option" indication in the packet capture embedded above for option 77. This is a known issue with wireshark due to the variation in the interpretation of option 77 ( see <https://ask.wireshark.org/questions/35332/dhcp-option-77-malformed-option/55006>). 
{: .notice--warning}


### DHCP Offer (DHCP Server to Router)

The cloudshark decode of the DHCP offer message in response to the request described above is shown below:  


<iframe src="https://www.cloudshark.org/captures/bba324c2b261/decode?frame=2&window=yes"
 style='width:100%; height: 200px;border: 5px solid #ccc; border-radius: 10px;'></iframe>
<div class="notice--info" style="margin:0em 0 !important;">
<div class="text-center">
    Interact above or <a href="https://www.cloudshark.org/captures/bba324c2b261">View on Cloudshark</a>
  </div>
</div>
&nbsp;  
&nbsp;  
 
While there are multiple options that the Server responds with, there are certain options/fields in particular that the IOS-XR ZTP infrastructure utilizes to determine the location of the script/config to download and the routing required to reach the web server. These are:

*  **Option 67 (bootfile-name)**: As part of 6.2.25, we support processing option 67 to glean the location of the script/config to download. When this option is used (as shown by the yellow (<mark>&nbsp;&nbsp;</mark>) highlight), the filename is NOT set in the BOOTP flags ( expand the Bootp flags in the DHCP offer capture above to see this), instead we see a separate option-67 field.. **IOS-XR will accept this and download the target file based on it. It will parse the option-67 bootfile-name to identify the server IPv4 address and filename separately.**

*  **Router (Option 3)**: As explained above, IOS-XR uses the filename to parse the server IPv4 address and combines it with the Router IP (option 3) as the gateway to install a static route towards the server IP during the ZTP process. 


The remaining two captures in <https://www.cloudshark.org/captures/bba324c2b261>  identify the DHCP request/reply exchange to wrap up the complete exchange between the router and the DHCP server.  




# DHCPv6 using Option 59

Post IOS-XR release version 6.2.25, IOS-XR ZTP also supports explicitly setting the bootfile-name (option 67) in place of the BOOTP filename explained above.

For this purpose we use the the DHCPv4 server configuration located [here](https://github.com/ios-xr/ztp-pcap/blob/master/6225/isc-dhcp-server-config/dhcpdv4-bootfile-name.conf).

The config is reproduced here for the reader's convenience:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
option dhcp6.user-class code 15 = string;
option dhcp6.bootfile-url code 59 = string;
option dhcp6.name-servers 2001:420:210d::a;
option dhcp6.domain-search "cisco.com";
option dhcp6.fqdn code 39 = string;

<span style="background-color: #98FF98">option dhcp6.vendor-class code 16 = { integer 32, integer 16, string };</span>
ddns-update-style none;

# option definitions common to all supported networks...
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;

default-lease-time 600;
max-lease-time 7200;

shared-network 2001-dba-100 {
  subnet6 2001:dba:100::/64 {
    # Range for clients
    range6 2001:dba:100::10 2001:dba:100::30;
    # Range for clients requesting a temporary address
    range6 2001:dba:100::/64 temporary;

    option dhcp6.name-servers 2001:dba:100::1;
    option dhcp6.domain-search "cisco.local"; 

    if exists dhcp6.user-class and substring(option dhcp6.user-class, 2, 4) = "iPXE" {
      option dhcp6.bootfile-url = "http://[2001:dba:100::1]/ncs5k-mini-4";
    } else if exists dhcp6.user-class and substring(option dhcp6.user-class, 0, 10) = "exr-config"     {
      <span style="background-color: #FDD7E4">if substring(option dhcp6.client-id, 1, 5) = 02:00:00:00:09 {</span> 
        <span style="background-color: #FDD7E4">if substring(option dhcp6.client-id, 6, 99) = 46:47:45:31:39:34:37:31:34:51:53:00 { </span>
          <span style="background-color: #98FF98">if substring (option dhcp6.vendor-class, 43, 99) = "NCS-5508" {</span>
            <mark>option dhcp6.bootfile-url = "http://[2001:dba:100::1]:9090/scripts/exhaustive_ztp_script.py";</mark>
          }
        }
      }
    }
  }
}
</code>
</pre>
</div>



## Packet Captures


All the relevant packet captures for this scenario have been shared on cloudshark: <https://www.cloudshark.org/captures/eeedef4dd779> and are embedded below.

>You can also find the corresponding .pcap file in the github repo here: <https://github.com/ios-xr/ztp-pcap/blob/master/6225/pcap/dhcpv6.pcap>



### DHCPv6 SOLICIT (Router to DHCPv6 Server)

Take a look at embedded cloudshark decode of the DHCPv6 Solicit packet.

<iframe src="https://www.cloudshark.org/captures/eeedef4dd779/decode?frame=1&window=yes"
 style='width:100%; height: 200px;border: 5px solid #ccc; border-radius: 10px;'></iframe>
<div class="notice--info" style="margin:0em 0 !important;">
<div class="text-center">
    Interact above or <a href="https://www.cloudshark.org/captures/eeedef4dd779">View on Cloudshark</a>
  </div>
</div>
&nbsp;  
&nbsp;  

 
The DHCPv6 Solicit packet sent by the router encodes the following information:

*  **Client Identifier (option 1)**: In DHCPv6 specification - [RFC 3315](https://www.ietf.org/rfc/rfc3315.txt) expects the client identifier to contain the DUID of the requesting device. The RFC also goes into further details on what the DUID must look like : [DUID](https://tools.ietf.org/html/rfc3315#page-19). Based off this, IOS-XR uses the following structure for the DUID field:  

   *  **DUID Type**: There are 3 DUID types defined in the RFC above, and IOS-XR uses type 2 = *"Vendor-assigned unique ID based on Enterprise Number"*. This is encoded as a two type field in hex: `00:02`
   *  **Enterprise Number**: [The Cisco IANA Enterprise Number is 9](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers). This is the first 4 bytes within the option and as part of the hex string:  `00:00:00:09`
      This unique identification allows complete automation of the device bringup (image and config/script download without ever logging into the device). In the DHCP server configuration above, this match is identified in red.  
   *  **Identifier**:  This field contains the encoded Serial Number of the device in hex. In the above example, it is: `46:47:45:31:39:34:37:31:34:51:53:00` which translates to `FGE194714QS` in ascii.

    Thus, the resultant field is `00:02:00:00:00:09:46:47:45:31:39:34:37:31:34:51:53:00`.
    The classification against this field is highlighted in red (<span style="background-color: #FDD7E4">&nbsp;&nbsp;</span>) in the DHCPv6 server config above.

*  **vendor-class (option 16)** : This option encodes the following information in this example:

          
   `Enterprise-Number PXEClient:Arch:00009:UNDI:003010:PID:NCS-5508`
     
   and is identical in the iPXE and ZTP phases. This can be broken down into:
   
    *  **Enterprise Number**:  [The Cisco IANA Enterprise Number is 9](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers). This is the first 4 bytes within the option and is encoded as an integer 32 in the option.
    *  **The type of client**: e.g. `PXEClient` 
    *  **The architecture of The system (Arch)**: e.g.: `00009` Identify an EFI system using a x86-64 CPU
    *  **The Universal Network Driver Interface (UNDI)**: e.g.: `003010` (first 3 octets identify the major version and last 3 octets identify the minor version)
    *  **The Product Identifier (PID)**: e.g.: `NCS-5508`. If you see the above dhcpv6 server config, the part highlighted in green (<span style="background-color: #98FF98">&nbsp;&nbsp;</span>) is used to extract and classify against the Platform ID (PID).
         
   
    
*  **User Class (Option 15)**:  This option is used by the client (router) to identify the current stage of operation. There are two provisioning stages :
    * **IPXE**:  This is the image download phase and is triggered when the device is brought up without an image or is forced into the iPXE mode in the BIOS. When the network device is performing iPXE, the user-class information encodes the string:  `iPXE`
    * **ZTP**:  This is the stage during which the network device expects to download the configuration/provisioning script that it should apply/execute. During this phase, the user-class information encodes the string: `exr-config`
    


### DHCPv6 Advertise (DHCP Server to Router)

The cloudshark decode of the DHCP offer message in response to the request described above is shown below:  


<iframe src="https://www.cloudshark.org/captures/eeedef4dd779/decode?frame=2&window=yes"
 style='width:100%; height: 200px;border: 5px solid #ccc; border-radius: 10px;'></iframe>
<div class="notice--info" style="margin:0em 0 !important;">
<div class="text-center">
    Interact above or <a href="https://www.cloudshark.org/captures/eeedef4dd779">View on Cloudshark</a>
  </div>
</div>
&nbsp;  
&nbsp;  
 
For DHCPv6, the primary option that the router needs from the Server is:

*  **Boot File URL (Option 59)**: This Option encodes the Boot file URL that is sent back to the router in response. **IOS-XR will accept this and download the target file based on it. This file may be a script or config and will be executed/applied accordingly.**



The remaining two captures in <https://www.cloudshark.org/captures/eeedef4dd779>  identify the DHCPv6 request/reply exchange to wrap up the complete exchange between the router and the DHCPv6 server.  


Hopefully, this blog is useful to anyone looking to deploy ZTP with IOS-XR. Do reach out if there are any concerns and we'll do our best to help out.
{: .notice--success}
