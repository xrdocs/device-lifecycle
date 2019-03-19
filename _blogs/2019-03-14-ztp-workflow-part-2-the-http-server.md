---
published: true
date: '2019-03-14 14:21 -0700'
title: ZTP Workflow Part 3 The HTTP Server
author: Patrick Warichet
excerpt: Part 3 the HTTP server
tags:
  - iosxr
  - cisco
  - linux
  - ZTP
  - HTTP
position: hidden
---
## Introduction

In this third blogs, we will go over the details of the HTTP/HTTPS server configuration. You can use any HTTP(S) server as long as they provide the "Content-Length" field in there response to the initial GET request from the client. The ZTP process will reject files of unknown length or file greater than 100MB. 

## Client Headers
IOS-XR devices include several information as part of the initial HTTP(S) request, a list of theses options are included in the table below. All options are sent in UTF-8

| Header          | Value                  | Example     |
|-----------------|:-----------------------|:------------|
| User-Agent      | HTTP Client            | curl/7.37.1 |
| X-cisco-arch    | CPU architecture       | x86_64      |
| X-cisco-uuid    | n/a                    | n/a         |
| X-cisco-oper    | Operational mode       | exr-config  |
| X-cisco-platform| Product Identification | NCS-5001    |
| X-cisco-serial  | Serial Number          | FOC1947R143 |

The information provided in the header can be used in place of the DHCP options. It allows the user to simplify the DHCP server configuration and only provide a generic link (e.g.: a PHP script) based on a common option provided by IOS-XR devices (e.g.: option 77/15). The script specific to the device is provided by dissecting the fields in the HTTP header as illlustrated in the following example

### Example1 Using the HTTP header with PHP

The following example requires an apache server with PHP installed as a module
The script parse the different fields of the headers, extract the Platform ID, the serial number and the operation mode and return a script  matching the serial number of the device.

```php
<?php
$headers = apache_request_headers();
foreach ($headers as $header => $value) {
  if (strpos($header, 'cisco-serial') !== false) {
    $serial = str_replace ("UTF-8''", "", $value);
  }
  if (strpos($header, 'cisco-platform') !== false) {
    $platform = str_replace ("UTF-8''", "", $value);
  }
  if (strpos($header, 'cisco-oper') !== false) {
    $operation = str_replace ("UTF-8''", "", $value);
  }
}
if ($operation == "exr-config" && $serial !== "") {
  $file_location = $_SERVER["DOCUMENT_ROOT"] . "/$platform/scripts/$serial.sh";
  if (file_exists($file_location)) {
    header ($_SERVER["SERVER_PROTOCOL"] . " 200 OK");
    header ("Accept-Ranges: bytes");
    header ("Content-Type: application/octet-stream");
    header ("Content-Length:".filesize($file_location));
    readfile ($file_location);
    die ();
  } else {
    die ("Error: File not found.");
  }
} else {
  die ("Error: Device is not in ZTP mode or device serial number not found");
}
?>
```
The script can be called using a generic DHCP configuration valid for all IOS-XR devices, removing the need of creating host specific sections in the DHCP server.

```
class "exr-config" {
   match if exists user-class and option user-class = “exr-config”;
}

pool {
     allow member of "exr-config";   
     range 192.168.0.30 192.168.0.254;
     option default-url = "http://192.168.0.22/php/ztp-generic.php";
     }
  }
```

## Server Headers

The HTTP server must include the content length in 

## Server Configuration

### HTTP Server

### HTTPs Server

## Using RESTConf
