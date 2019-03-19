---
published: false
date: '2019-03-14 14:21 -0700'
title: ztp-workflow-part-2-the-http-server
---
## Introduction

In this third blogs, we will go over the details of the HTTP and HTTPS server.

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

The information provided in the header can be used in place of the DHCP options. It allows the user to simplify the dhcp server configuration and only provide a generic link (e.g.: a PHP script) based on a common option provided by the devices (e.g.: option 77/15). The script specific to the device is provided by disecting the fields in the header as illlustrated in the following example

### Example1 Using the HTTP header with PHP


```

<?php
$headers = apache_request_headers();
foreach ($headers as $header => $value) {
  if (strpos($header, 'cisco-serial') !== false) {
    $serial = str_replace ("UTF-8''", "", $value);
    error_log ("Serial: $serial", 0);
  }
  if (strpos($header, 'cisco-platform') !== false) {
    $platform = str_replace ("UTF-8''", "", $value);
    error_log ("Platform: $platform", 0);
  }
  if (strpos($header, 'cisco-oper') !== false) {
    $operation = str_replace ("UTF-8''", "", $value);
    error_log ("Operation: $operation", 0);
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
    // error_log ("Error: File $serial.sh not found.", 0);
    die ("Error: File not found.");
  }
} else {
  die ("Device is not in ZTP mode or device serial number not found");
}
?>
```


## Server Headers

The HTTP server must include the content length

## Server Configuration

### HTTP Server

### HTTPs Server

## Using RESTConf
