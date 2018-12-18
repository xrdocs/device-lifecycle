---
published: true
date: '2016-10-17 11:46 -0700'
title: IOS-XR users and groups inside Linux
author: Patrick Warichet
tags:
  - iosxr
  - cisco
  - linux
excerpt: How IOS-XR users and groups maps to Linux
---

{% include toc icon="table" title="IOS-XR: Linux users and groups" %}  

## XR and Linux Users

By default, any user created inside XR is automatically replicated including that user's password inside Linux using a unique UID and GID.

This allows basic access into the Linux shell for all XR configured user, the administrator can create multiple users directly from the XR console if desired.

Inside Linux, nine special groups are created by default, each of these groups maps to one of the default XR groups. When the administrator creates a user belonging to one of the default XR group, that user get replicated inside Linux and added to that special Linux group. In addition, XR users that are member of certain default group are granted root access to Linux when they issue the "bash" or "run" command (see table below).

| **XR Group**| **Linux Group**| **GID**| **Role**| **Access Linux from XR**
| cisco-support| cisco-support| 1000| Cisco support personnel tasks| n/a (add-on group for root-lr users)
| maintenance| maintenance| 1001| | Yes
| netadmin| netadmin| 1002| Network administrator tasks| No
| provisioning| provisioning| 1003| | Yes
| retrieve| retrieval| 1004| | No
| root-lr| root-lr| 1005| Secure domain router administrator tasks| Yes
| n/a| root-system| 1006| System-wide administrator tasks| Compatibility with previous IOS-XR
| serviceadmin| serviceadmin| 1007|	Service administration tasks| No
| sysadmin|	sysadmin| 1008|	System administrator tasks| No
| operator|	operator| 37| Operator day-to-day tasks (demo)| No

In the following example, the administrator creates a user "test10" member of the maintenance group, which allow the user to enter the Linux shell as root (uid/gid: 0) via the run command, the user "test10" (uid: 1010) is member of the group "test10" (gid:1019) and secondary group maintenance (gid:1001)

```
RP/0/RP0/CPU0:pod-rtr(config)#username test10 group maintenance
RP/0/RP0/CPU0:pod-rtr(config)#username test10 password test10
RP/0/RP0/CPU0:pod-rtr(config)#commit
RP/0/RP0/CPU0:pod-rtr(config)#end
RP/0/RP0/CPU0:pod-rtr#exit
...
Username: test10
Password:
RP/0/RP0/CPU0:pod-rtr#bash
Tue Mar 29 03:06:41.759 UTC

[xr-vm_node0_RP0_CPU0:~]$id
uid=0(root) gid=0(root) groups=0(root)
[xr-vm_node0_RP0_CPU0:~]$id test10
uid=1010(test10) gid=1019(test10) groups=1019(test10),1001(maintenance)
```

## Gaining Root Privilege

Inside Linux the file /etc/sudoers only allows root to do everything on any machine as any user ```"root ALL=(ALL) ALL"```. No other user can gain root privilege by default, The administrator will have to modify the /etc/sudoers as root to allow other users to gain root access via the sudo command.

These measures ensure that only the users with access to the "run" or "bash" command can create initial users in the Linux shell and provide sudo access.

A common practice is to allow all members of the sudo group (GID: 27) root privileges. This is done by un-commenting the line ```"#%sudo ALL=(ALL) ALL"``` as root in the "/etc/sudoers" file and add users to the sudo group.

```
[xr-vm_node0_RP0_CPU0:~]$usermod -a -G sudo test10
[xr-vm_node0_RP0_CPU0:~]$id test10
uid=1010(test10) gid=1019(test10) groups=1019(test10),27(sudo),1001(maintenance)
```

After the line has been modified any user member of the sudo group can gain root privilege using the sudo command. Sudo logs each command, providing a clear audit trail of who did what, Sudo uses timestamp files to implement a "ticketing" system. When a user invokes sudo and enters their password, they are granted a ticket for 5 minutes. for more information on sudoers visit [sudo main page](https://www.sudo.ws/sudo.html "sudo main page")

---
**Update:** Since 6.1.1 All members of the the sudo group can gain root privilege using "sudo" by default.

---
