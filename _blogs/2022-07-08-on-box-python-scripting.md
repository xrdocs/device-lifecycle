---
published: true
date: '2022-07-08 16:10 -0500'
title: On-Box Python Scripting
tags:
  - iosxr
  - cisco
position: hidden
author: Eliot Pickhardt
excerpt: Automation using python scripts
---
{% include toc %}
# Programmability of IOS XR
##### Eliot Pickhardt
Being a network engineer no longer requires poring over CLI commands for hours on end, ensuring every detail conforms to the requirements, just for a link to break, causing even more work. Using the programmability capabilities of IOS XR, you eliminate much of the tedious router-by-router configuration by manipulating the existing data models automatically. There are many ways to tap into the automated potential of IOS XR, including ZTP, Automation Scripts, and gNMI.
## Automation Scripts
Automation scripts are another way to leverage IOS XR to work for you. These are mainly Python scripts that run on-box. These scripts can work in four different ways to aid the configuration and maintenance of your network. 

**Config scripts**: These scripts run automatically every time a configuration change is committed. They are useful to ensure that a commit doesn’t go against any network rules, and can take action or throw errors if rules are broken.

**Exec Scripts**: These scripts are run manually, but they can dramatically decrease the work required for configuration or other operational tasks.

**Process Scripts**: These scripts run continually once manually activated. They perform typical checks and will exit upon a preconfigured condition. They daemonize normal system monitoring.

**EEM Scripts**: These are event-driven scripts, and can be configured to run under a variety of conditions, such as a preconfigured timing interval or in response to a system condition. They operate similarly to Exec scripts in that they can aid in configuration or system maintenance. 

> Note: As of IOS XR release 7.5.1, EEM scripts are not supported


### Using Syslog in Python Scripts
All types of on-box python scripts have access to the logging capabilities of IOS XR. Using the `cisco.script_mgmt` library, we can import `xrlog`. This allows us to send information to syslog within our scripts, using the `syslog = xrlog.getSysLogger('name_of_script')` function. From here, we can print [all levels of syslog](https://www.cisco.com/c/en/us/td/docs/routers/access/wireless/software/guide/SysMsgLogging.html#wp1054858) information using the `syslog.<level>('message')` syntax.  

## Exec Scripts

Exec scripts represent the most basic on-box scripting available within IOS XR. These scripts provide a way for the network managers to manually deploy programs that simplify their work as a whole, including automating the configuration process. In order to do this, we must leverage the `iosxr.xrcli.xrcli_helper` library. Being able to commit configurations from inside a python script enables us to streamline many of the repetitive CLI configurations we complete. Exec scripts will commonly use the `xr_apply_config_string(configuration)` function from this library. This function takes a CLI configuration command as input, and then commits the provided configuration. However, the argument of this function is a *single-line* configuration, meaning we must make use of a few tricks to issue complex configurations.  

Using string formatters allows us to adjust the configuration to a command-line argument or an environment variable:
```py
xrcli_helper.xr_apply_config_string("interface %s" %interface_name)
```
where `%s` indicates a string insertion.  

A second tool we can use is escape characters. Specfically, the `\n\r` combination will provide similar function to pressing "Enter" while using the CLI. This allows us to dramatically increase the complexity of commands used within this method.
```py
xrcli_helper.xr_apply_config_string("interface TenGigE0/0/0/1 \n\r ipv4 address 10.0.0.2 \n\r no shutdown")
```
Using a combination of these two techniques enables us to maximize the potential of this function and Exec scripts overall.  

## Config Scripts
As mentioned, config scripts are the best way to ensure that a commit doesn’t violate any existing rules for the network. Each config script should be relatively specific in its use (ie, regarding one protocol). Breaking down the general form of these scripts will help us understand exactly how they work.

`xr.register_validate_callback(path, callback_function)`


This function is required to be called in each config script. 

The first argument to this function, `path`, determines when the config script should be run. The script should only be called when a commit related to the script is pushed in order to reduce power usage. The `path` is a YPath from the data model the script uses to check for valid configuration. Specifically,  this argument needs to be a schema path, which is a YPath that doesn’t point to a specific instance of an item. This means that no keys in the path are specified. 

There are a number of limitations on the available models that can be used for config scripts. Only XR-native YANG models are supported (no UM, IETF, OC). Also, in order for the callback function to properly access the configuration data, the path needs to be from a “cfg” model, not an “oper” model. 

The second argument to this method, `callback_function`, is what will be run whenever a relevant commit is pushed to the configuration.

##### `callback_function`
The callback function is the “meat” of the config script. The signature of the function needs to be:
```py
def cb_fn_name(root):
    #function goes here
```

where `root` is the root node.

This function retrieves specific nodes from the YANG model in order to check configuration status, including containers, leaves, and leaf-lists. In order to do this, we can use either the `<node>.get_node(path)` or `<node>.get_list(path)`. For each of these methods, the path is a specific instance of a YPath, called a data path. Unlike the schema path mentioned in the previous section, the keys of the path must be included to find a specific node or list. Intuitively, `get_node` can be used to get leaf or container nodes, while `get_list` should be used to obtain leaf-list types. 


If the node retrieved is a leaf node, you can use the `.value` attribute to retrieve the associated data. Meanwhile, leaf-lists can be iterated through to get the leaf nodes within them. 

After the relevant data has been retrieved from the models, the necessary validation can occur. In this stage, you can send messages to the syslog, as well as having a few options to take with configurations that don’t comply with the desired standards. The first route to take is to generate warnings in the syslog.

```py
syslog.warning("Something is wrong!")
```

Next, you can block the commit by generating errors from the xr library. This will also send a custom error message to the user. 

```py
xr.add_error(curr_node, "Something is wrong! Can't commit changes")
```

Finally, you can change configuration data automatically using the `curr_node.set_node(path, data)` or `curr_node.set(data)` methods. The former allows you to set the data of the current node or any child node relative to the current node, while the latter works for the current node only. 

```py
curr_node.set_node("/leaf", data_to_set)

curr_node.set(data_to_set)
```

Using these methods will allow the commit to pass with the changes.

### Process Scripts
Process scripts are the best way for a user to automatically monitor operational data within IOS XR. Since process scripts run continuously by nature, we must register them with AppMgr for them to run. Information about how to correctly set up process scripts can be found [here](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-5/programmability/configuration/guide/b-programmability-cg-asr9000-75x/process-scripts.html). 

Process scripts begin with a typical Python `if __name__ == "__main__":` statement. In this statement, there must be an infinite loop (`while(1)`), which can call any necessary helper functions. Many process scripts also utilize the `time` library, which allows the script to wait for a number of seconds (or minutes) at the end of the script before running again. This is helpful in saving power, since script execution is suspended during this time. 

When creating a process script, a common practice is to use a NETCONF RPC to retrieve and edit operational data. We can import `NetconfClient` from the `iosxr.netconf.netconf_lib` in order to access an RPC. All [NETCONF operations](https://en.wikipedia.org/wiki/NETCONF#Operations:~:text=SNMP%20modeling%20language.-,Operations,-%5Bedit%5D) are available with this rpc with the following syntax:
```py
nc = NetconfClient(debug=True)
nc.connect()
nc.<operation>(file=None, request=None)
nc.close() #always close the netconf session when you are done
```
In order to find the correct piece of data, we should use a string representing an XML filter. The structure of this string is:
```py
<container1 xmls="http:cisco.com/ns/yang/Cisco-IOS-XR-{namespace of YANG model}>"
    <container2>
        <key-to-container2>KeyData</key-to-container2>
            <leaf-name/>
    </container2>
</container1>
```
where the leaf contains the data we wish to monitor/manipulate. This filter string should be passed to the `request` argument of the NETCONF function.

We can use the reply from the operation (`nc.reply`) and details about the filter path to retrieve the desired data from here. The reply will be in XML, so we will have to employ some parsing strategies to extract the specific data. Examples of how to do this can be found in the videos at the end of this section. 

Similarly to the config scripts, we can raise syslog errors (among other options) if operational data doesn't match requirements, or a change in the system has been detected. 

### EEM Scripts
As EEM scripts are not currently supported, there is no script-writing guide available. 

**Full script examples can be found at the xr_python_scripts [GitHub repository](https://github.com/CiscoDevNet/xr-python-scripts)**