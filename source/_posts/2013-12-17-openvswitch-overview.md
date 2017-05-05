---
layout: post
title: 'OpenvSwitch - overview'
date: 2013-12-17 15:13
comments: true
categories: 
tags:
	- SDN
	- Network
	- OpenvSwitch
---
This article show what the system do when we install the OpenvSwitch

####Environment####
- Three PCs.
- One for openvswitch (with a 4-port ethernet card).
- Two for hosts.
- OVS version 1.9

<!--more-->

![test.png](http://user-image.logdown.io/user/415/blog/415/post/167510/x1arC8nSTiOAQ0AoLtjj_test.png)

####kernel module####
`insmod datapath/linux/openvswitch.ko`

When we load the **openvswitch**'s kernel module, it will register four **generic netlink** event including
`datapath`, `vport`, `flow` and `packet`.

![擷取1.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/a9o3mQ2iR2GrYSRMKuIN_%E6%93%B7%E5%8F%961.PNG)

In the **datapath.c**, we can see those four **generic netlink**  type.

![2.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/Tt1PPwiHSMiBWmKiztJM_2.PNG)

Take the **vport** for example, there're four command we can excute via this **netlink** type.
If we want the kernel to create a new port, we can send the **vport** type **netlink** with the command **OVS_VPORT_CMD_NEW**
, and the command handler (**doit**) ovs_vport_cmd_new will be excuted to create the new vport.

![擷取.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/eHZ7vqScSjCAaFJMDiIn_%E6%93%B7%E5%8F%96.PNG)

####ovs-vswitchd####
` ovsdb-server ...`
` ovs-vswitchd --pidfile --detach `

- First, the `ovsdb-server` will start a database daemon, In addition, there're some user-space tool will work with it, like **ove-vsctl**, **ovs-ofctl**..etc.
- The user-space process **ovs-vswitchd** play a importmant role about **openflow** in OpenvSwitch.
It will parse the openflow protocol and handle it (you can use the keyword **ofproto** to find the resource about it)


ovs-vswitchd:
- Process openflow messages
- Manage the datapath (which actually in kernel space)
- Maintain two flow table (exactly flow & wildcard flow)

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/A5R1wlMlQMGHHmAMjURg_%E6%93%B7%E5%8F%963.PNG)

####Adding bridge####
`ovs-vsctl add-br br0`

When we excute the `ovs-vsctl`, it will send a command to **ovsdb** and the DB will store this information.
After that, the **ovsdb** will pass the command to **ovs-vswitchd**, and the **ovs-vswitch** send the **netlink**  with **datapath** type to the kernel.
Since we have installed the kernel module before, the datapath will receive the netlink and excute the corresponding command handler.
In this case, it will excute **ovs_dp_cmd_new**.
Finally, the **datapath** will be created and it will be managed by **ovs-vswitchd**.

datapath:
- Maintain one flow table (exactly flow) **This study is based on the OVS v1.9**
- Act as the software switch (look up flow, forward the packet)

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/22cYSkNQQwmksjbBcDPq_%E6%93%B7%E5%8F%963.PNG)

####Adding vports####
`ovs-vsctl add-port br0 eth1`

Like the above discussion about **datapath**, **ovs-vswitchd** send the **netlink** to the kernel.
In the command handler **ovs_vport_cmd_new**.

1.Find the the **struct net_device** object in the kernel by the user typing interface name (**eth1**)
2.Modify the receive_handler of that **net_device** to the OpenvSwitch's packet handler.

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/jdtSnR6SbCZRX2QcwTqQ_%E6%93%B7%E5%8F%963.PNG)

####Set-Controller####
`ovs-vsctl set-controller br0 tcp:xxx.xxx.xxx.xxx:6633`

- Set the controller setting and it will be done in **ovs-vswitchd**.

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/Lioqm31mTWqVrFUAkbTZ_%E6%93%B7%E5%8F%963.PNG)

##In the following example, we use a simple case to explain how the ping works


####Target command####
`hostA ping hostB`

We devide the picture into two parts by the red line.

**Upper Part**
- This part show the physical view of thie case.
- The middle PC has installed the Ubuntu 12.04 and OVS 1.9.
- The left PC connect to the OVS's nic **eth1**
- The right PC connect to the OVS's nic **eth2**

**Lower Part**
- This part show the system view of the switch PC (middle one)
- We use the dash-line to separate the user-space and kernel-space.

**Analysis**

After the OVS receives the ICMP packet from the left PC.
What will happen about OVS?

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/iQ4NzZPtTEyHzA4XyXln_%E6%93%B7%E5%8F%963.PNG)


1. The NIC **eth1** receives the ICMP packet.
2. Call the **receive_handler** to handler this ICMP packet.
3. Do `flow_lookup`, it will look up the flow table maintained by the kernel-space. All the flow entry in this table is **exactly** flow entry, which means there're no any wildcard.
This architecture will speed up the look-up since we don't need to consider the wildcard field.
In the OpenvSwtich, it use the **struct sw_flow_key** to present a **exactly flow**.
4. If we find the flow entry in the flow table, excute its **flow actiojn**.
5. Otherwise, we need the help from controller. so the **datapath.ko** will send this flow to the user-space via the f unction **upcall**
(actually, it's a netlink message)

What will happen when the **ovs-vswitch** receive the flow from the kernel-space.

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/itUv393WQbS2dl34nKjG_%E6%93%B7%E5%8F%963.PNG)

- Both **exactly matching flow** and **wildcard matching flow** are stored in the user-space (by Openflow protocol).
- Since the **exactly matching** has high priority than **wildcard matching**, we need to lookup the **exactly macthing flow table** first.
- Look up the flow entry in the user-space by **exactly matching**, if we find it, send **two** netlink message to the kernel (we will discuss these two nelitnkj message later)
- Otherwise, look up the flow entry by **wildcard matching**, if we find it, generate a corrsponding exactly flow entry and send **two** netlink message to the kernel.
- If we can't find any flow entry in the flow-table, we issue a **Packet_In** to the controller.


After the kernel-space receive those **two** netlink message which sending from user-space.
1. Excute the **flow_actiojn** about that flow entry.
2. Insert that **exactly mactching flow** into the kernel's flow-table. That will create the cache for that connection  and crease the processing time for nect packets.

![擷取3.PNG](http://user-image.logdown.io/user/415/blog/415/post/167510/VJcdFvSAawgDpSoLDrVA_%E6%93%B7%E5%8F%963.PNG)

**Summary**

- There is a limitation about the size of flow table in kernel, it use the cache (exactly macthing) to speed up the look-up.For the recently activity connection, those packets can be handled quickly.
- The flow-table in the user-space is the same as the what the controller see. It support the wildcard matching. We can reduce the size of flow entries by wildcard matching but it will bring the overhaed for look-up


**MISC**
1. You can use the **ovs-dpctl dump-flows** to dump the flow table of kernel-space
2. This article is based on the OVS v1.9 and the architecture has some change after v1.11


