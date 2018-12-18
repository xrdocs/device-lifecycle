---
published: false
date: '2018-05-08 10:47 -0700'
title: Cable Management with ZTP
---
## Foreword
A while ago I wanted to refresh my knowledge of Python and get familar with JSON data structure. I decided to create a small project to help me verify that all the connections from and to newly inserted devices in my lab were correct and functional. The script in this blog may not be very Pythonesque :) as I was still learning Python but it does the job.

## Introduction
Cable Management is a a key element of provisioning a device, after installing the device, inserting the optics and cabling everything, we would like to know if everything has been setup correctly before starting the configuration process. In this blog I will go over the visualization part of the cable management, I will go over the verification process in part 2.

This example replies on LLDP to obtain information from the neighbors, LLDP provides the following usefull information:
1. The local/remote interfaces for each connection
2. The neighbor chassis identification
3. The neighbor device name
4. The neighbor device description (PID and SW version if the neighbor is IOS-XR)
5. The neighbor management IPv4/IPv6 address (if configured)

## Graph
Graph visualization is a way of representing structural information as diagrams of abstract graphs and networks. It has a wide range of application including database, bioinformatics software engineering and networking.

Their are multiple formats use to describe graphs: DOT, GXL, GRAPHML and multiple tools to generate and process graph. In my research, I found [Graphviz](https://www.graphviz.org/) and [Gephi](https://gephi.org/) to be the most used. Gephi used the [GEXF](https://gephi.org/gexf/format/) (Graph Exchange XML Format) language internally and includes a set of tools that can generate and/or process [various format](https://gephi.org/users/supported-graph-formats/), it also can be enhanced using plugins and found out that there is a [JSON export plugin](https://github.com/oxfordinternetinstitute/gephi-plugins/tree/jsonexporter-plugin/modules/JsonExporter).

### Edges and Nodes
For network graph, we define a node as a LLDP speaking device and an edge as an LLDP learned connection between 2 nodes.
The JSON format exported by Gephi can be interpreted in python as a dictionary of an edges list and a nodes list.
```
graph = {}
graph['nodes'] = []
graph['edges'] = []
```
A node or an edge is represented in Python using a dictionary that has some mandatory and some optional keys. I decided to organize all the optional parameters in a dictionary under the optional "attributes" key.
The "label" (name) and "id" (sequential number) keys are mandatory for both edges and nodes, for edges the "source" and "target" keys are also mandatory to define the origin and destination node of each edge.
Some of the key/value pairs in the "attribute" dictionary are not taken from the "show lldp neighbor" output but from the "show controller interface" output and can vary from platform to platform.

I included the Digital Optical Monitoring (DOM) parameters collected from the "show controller" command. Not all transceivers supports the DOM feature but if supported, it provides a good overview of the quality of the signal received on all the transceiver lanes. It also alow you to monitor the temperature and Vcc reported by the transceiver.

The "attributes" dictionary can be customized to include any command output you may find relevant.

The following JSON dictionay describes an example of a node and edge.

```
node = {
  "label": "",
  "id": "",
  "attributes": {
     "title": "",
     "chassis id": "",
     "system description": "",
     "mgmt address": {
       "IPv4": "",
       "IPv6": ""
     }
  }
}
```

```
edge = {
      "label": "",
      "source": "",
      "target": "",
      "id": "",
      "attributes": {
         "remote interface": "",
         "local interface": "",
         "title": "",
         "operational values": {
            "speed": "",
            "duplex": ""
            },
         "optics": {
            "media type": "",
            "vendor": "",
            "PN": "",
            "waveLength": "",
            "DOM": {
               "temp": "",
               "voltage": "",
               "lane 0": {
                   "Tx power": "",
                   "Rx power": ""
                },
                "lane 1": {
                   "Tx power": "",
                   "Rx power": ""
                 },
                 "lane 2": {
                    "Tx power": "",
                    "Rx power": ""
                 },
                 "lane 3": {
                    "Tx power": "",
                     "Rx power": ""
                 }
            }
         }
      }
   }
```
##Workflow
The python script can be downloaded and executed trough the ZTP process. The script will perform the following tasks:

1. Create a configuration file that enable all the relevant interfaces and activate LLDP
2. Apply the configuration
3. Iterate through the list of LLDP neighbors and creates the nodes and edges list
4. Populate the edges with the optics values
5. Save the graph on the local disk
6. Export the graph via HTTP POST to a web server

## Exporting the graph
I wanted to export the JSON graph file directly from the device to an apache web server using HTTP POST, apache support PUT and POST if you allow these metodods in the configuration of the virtual server but you still need a way to process the request. I create a simple PHP handler for this purpose. The PHP script simply copy the file to a specified direactory and use the name provided in the HTTP POST form "file_contents". It return the status of the operation in JSON format for easier processing.

```
<?php
$uploaddir = realpath('./datasources') . '/';
$uploadfile = $uploaddir . basename($_FILES['file_contents']['name']);

	if (move_uploaded_file($_FILES['file_contents']['tmp_name'], $uploadfile)) {
		$data = array("status"=>"success","output"=>$_FILES);
        echo json_encode($data);
	} else {
		$data = array("status"=>"error","output"=>$_FILES);
        echo json_encode($data);
	}
	//echo 'Here is some more debugging info:';
	//print_r($_FILES);
?>
```

IOS-XR comes standard with urllib, urllib2 and httplib which can be used to perform an HTTP POST request. Unfortunatly neither urllib nor httplib directly support mime type "multipart/form-data", 

```
def post_multipart(self, host, selector, fields, files):
  """
  Post fields and files to an http host as multipart/form-data.
  @param host: the hostname of the server to connect to.  e.g.: www.myserver.com
  @param selector: where to go on the host. e.g.: cgi-bin/upload.php or datasources/upload, etc..
  @param fields: a sequence of (name, value) elements for regular form fields.  For example:
  [("vals", "16,18,19"), ("foo", "bar")]
  @param files: a sequence of (name, file) elements for data to be uploaded as files. For example:
[ ("Erebus", open("/images/me.jpg", "rb")) ]
  @return: the server's response page.
  """
  content_type, body = ztp_script.encode_multipart_formdata(fields, files)
  h = httplib.HTTPConnection(host)  
  headers = {
  'User-Agent': 'python_multipart_caller', 'Content-Type': content_type
  }
  h.request('POST', selector, body, headers)
  res = h.getresponse()
  return res.read() 

def encode_multipart_formdata(self, fields, files):
  """
  @return: (content_type, body) ready for httplib.HTTP instance
  """
    
  BOUNDARY = '----------ThIs_Is_tHe_bouNdaRY_$'
  CRLF = '\r\n'
  L = []
  for (key, value) in fields:
    L.append('--' + BOUNDARY)
    L.append('Content-Disposition: form-data; name="%s"' % key)
    L.append('')
    L.append(value)
  for (key, fd) in files:
    file_size = os.fstat(fd.fileno())[stat.ST_SIZE]
    filename = fd.name.split('/')[-1]
    contenttype = 'application/octet-stream'
    L.append('--%s' % BOUNDARY)
    L.append('Content-Disposition: form-data; name="%s"; filename="%s"' % (key, filename))
    L.append('Content-Type: %s' % contenttype)
    fd.seek(0)
    L.append('\r\n' + fd.read())
  L.append('--' + BOUNDARY + '--')
  L.append('')
  body = CRLF.join(L)
  content_type = 'multipart/form-data; boundary=%s' % BOUNDARY
  return content_type, body
```
 


