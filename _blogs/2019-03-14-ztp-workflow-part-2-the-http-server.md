---
published: false
date: '2019-03-14 14:21 -0700'
title: ztp-workflow-part-2-the-http-server
---
## Introduction

In this third blogs, we will go over the details of the HTTP and HTTPS server.

## Client Headers
IOS-XR devices include several informations as part of the initial HTTP(S) request, a list of theses options are included in the table below. All options are sent in UTF-8

| Header          | Value                  | Example     |
|-----------------|:-----------------------|:------------|
| User-Agent      | HTTP Client            | curl/7.37.1 |
| X-cisco-arch    | CPU architecture       | x86_64      |
| X-cisco-uuid    | n/a                    | n/a         |
| X-cisco-oper    | Operational mode       | exr-config  |
| X-cisco-platform| Product Identification | NCS-5001    |
| X-cisco-serial  | Serial Number          | FOC1947R143 |

## Server Headers

The HTTP server must include the content length

## Server Configuration

### HTTP Server

### HTTPs Server

## Using RESTConf
