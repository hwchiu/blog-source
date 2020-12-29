---
title: '[Container Network Interface] Implement Your CNI In Golang'
keywords: 'Container,Network,Interface,golang,implement,program'
tags:
  - CNI
  - Network
  - Docker
  - Linux
  - Ubuntu
  - Golang
  - Kernel
  - Kubernetes
abbrlink: 18895
date: 2018-06-16 08:34:18
description: As we know, the kubernetes use the CNI to provide the network connectivity for its Pod unit and the cluster administrator can choose what kind of the CNI should be installed in the cluster. For example, if the only requirement is the overlay network, you can choose the flannel CNI and choose the calico CNI if you have the requirement of the BGP. In this post, we will learn how to write your own CNI in golang language. Actually, You can implement it with any language as you like.


---


# Preface

It's a series post about the Container Network Interface and you can find other posts below.
[[Container Network Interface] Bridge Network In Docker](https://www.hwchiu.com/introduce-cni-i.html)
[[Container Network Interface] CNI Introduction ](https://www.hwchiu.com/introduce-cni-ii.html)

In this post, I will show how to write your own CNI program.

Container Network Interface(CNI) can be implemented by any programming languages as you like.

You just to follow the interface and your program can be used for every infrastructure using the CNI for their network connectivity.

In this tutorial, I will use the `golang` to implement a simple CNI witch create a `Linux Bridge` in the host and connect the container and the host itself.

I have create a github repo for this tutorial and you can find it on [hwchiu CNI_Tutorial_2018](HTTPS://github.com/hwchiu/CNI_Tutorial_2018)


# Introduction
In order to help the develop to develop their own CNI, the `CNCF` had setup two projects for developers.

Those projects are based on the golang language and provide useful libraries for the developer to control the Linux network functions, such as `IP`, `netlink` and `network namespace`.

The [ContainerNetworking/CNI](HTTPS://github.com/containernetworking/cni) provides the basic function for CNI implementation in golang and you can see the introduction of that project in its README
>As well as the specification, this repository contains the Go source code of a library for integrating CNI into applications and an example command-line tool for executing CNI plugins. A separate repository contains reference plugins and a template for making new plugins.

The other project [ContainerNetwokring/Plugins](HTTPS://github.com/containernetworking/plugins) provides some basic network functions for your CNI and it can be divided into two types.
## Basic CNI
It provides some basic CNI, such as `Bridge`, `MacVlan`, `Host Device`.. And so on.
You can chain those CNI into your own CNI and combine those into a more powerful CNI.
## IPAM
IPAM (IP Address Management) provides some method to handle the IP/Route management. It provides `host-local`, `dhcp` and `static` three methods now.

In the `host-local`, you just need to provide a configuration file to describe what subnet/gateway you want to use and it will allocate a unused IP address from that subnet for your CNI.
And the `dhcp` will runs a `DHCP` client in each container and send a `dhcp request` to get a IP address from the `dhcp` server.

In this tutorial, we will implement a `bridge` CNI and explain those functions step by step.

# Before We Start
Before we start to implement the CNI, we must know the `interface`/`specification` of the `CNI`.

1. Your CNI will be invoked when the container is `ready to create or has been terminated`.
- Allocate resources for the container, including the `IP address` and the network connectivity.
- Remove all resources you allocated before when a container has been terminated.
2. The caller will pass the following information into your CNI program
- Command (What kind of the event you should care)
    - ADD
    - DELETE
    - VERSION
- ContainerID (The target ContainerID)
- NetNS (THe network namespace path of the container)
- IFNAME (The interface name should be created in the container)
- PATH (The current working PATH, you should use it to execute other CNI)
- STDIN (The configuration file of your CNI)

# Step By Step
For each step, you can find a corresponding folder in my github repo and there's all golang files for each steps.

## Step1
First, we need to provide two function for `ADD` and `DELETE` event which is used to allocate/recycle resource when the container has been start/terminated.

We use the framework provided by the The [ContainerNetworking/CNI](HTTPS://github.com/containernetworking/cni) and it will encapsulate


```go=
Package main

Import (
    "github.com/containernetworking/cni/pkg/skel"
    "github.com/containernetworking/cni/pkg/version"
)

func cmdAdd(args *skel.CmdArgs) error {
    return nil
}

func cmdDel(args *skel.CmdArgs) error {
    return nil
}

func main() {
    skel.PluginMain(cmdAdd, cmdDel, version.All)
}
```


In this framework, it encapsulates all information we need into a predefined type `skel.CmdArgs`

```go=
type CmdArgs struct {
    ContainerID string
    Netns string
    IfName string
    Args string
    Path string
    StdinData []byte
}

```
Use the `go build` to build the binary and assume our execution file  is `example` and then we should provide a basic configuration which should contains useful information for our CNI.
Maybe we call the file `configuration` its contents looks like
```json
{
	"name": "mynet",
	"BridgeName": "test",
	"IP": "192.0.2.1/24"
}
```

Now, We can use the following command to execute our CNI program.
```shell=bash
sudo CNI_COMMAND=ADD CNI_CONTAINERID=ns1 \
CNI_NETNS=/var/run/netns/ns1 CNI_IFNAME=eth10 \
CNI_PATH=`pwd` \
./example < config
```

For that go CNI framework, those infromation should be passed by the `environement` and we can get that from the `CmdArgs`.

Actually, we have done the basic CNI program but it does nothing.

A good CNI should make a container network connectivity and assign a valid IP address to the container and we will do that in the foloowing tutorial.


## Step 2
Now, we will create a linux bridge for the container and the logical flow looks like
1. Read the bridge information from the config.
2. Get the bridge name we want to use.
3. Create the bridge if it doesn't exist in the system.

Since the frametwork store the config content in the `CmdArgs` object as a `[]byte` form. we should create a `structure` to decode those `[]byte` data.

```go=
type SimpleBridge struct {
    BridgeName string `json:"bridgeName"`
    IP    string `json:"ip"`
}
```
and decode the config content in the `CmdAdd` function.

```go=
func cmdAdd(args *skel.CmdArgs) error {
	sb := SimpleBridge{}
	if err := json.Unmarshal(args.StdinData, &sb); err != nil {
		return err
	}
	fmt.Println(sb)
```

There're many ways for creating the Linuxu Bridge, we can use the system commadn `brctl addbr` via the `os.Exec` or use the `netlink` to create.

We choose the `netlink` method here since the `os.Exec` is too easy for developer.

First, we should import the `netlink` package `"github.com/vishvananda/netlink"`
and we will use the type `netlink.Bridge` to describe the bridge we want.

In the following example, we will do three things.
1. Prepare the netlink.Bridge object we want.
2. Create the Bridge
3. Setup the Linux Bridge.

```go=
	br := &netlink.Bridge{
		LinkAttrs: netlink.LinkAttrs{
			Name: sb.BridgeName,
			MTU:  1500,
			// Let kernel use default txqueuelen; leaving it unset
			// means 0, and a zero-length TX queue messes up FIFO
			// traffic shapers which use TX queue length as the
			// default packet limit
			TxQLen: -1,
		},
	}

	err := netlink.LinkAdd(br)
	if err != nil && err != syscall.EEXIST {
		return err
	}

	if err := netlink.LinkSetUp(br); err != nil {
		return err
	}
```

Now. The `CmdAdd` function should look like below.
```go=
func cmdAdd(args *skel.CmdArgs) error {
    sb := SimpleBridge{}
    if err := json.Unmarshal(args.StdinData, &sb); err != nil {
        return err
    }
    fmt.Println(sb)

    br := &netlink.Bridge{
        LinkAttrs: netlink.LinkAttrs{
            Name: sb.BridgeName,
            MTU:  1500,
            // Let kernel use default txqueuelen; leaving it unset
            // means 0, and a zero-length TX queue messes up FIFO
            // traffic shapers which use TX queue length as the
            // default packet limit
            TxQLen: -1,
        },
    }

    err := netlink.LinkAdd(br)
    if err != nil && err != syscall.EEXIST {
        return err
    }

    if err := netlink.LinkSetUp(br); err != nil {
        return err
    }
```
Use the aforementioned command to call the binary again and you should see the linux bridge `test` has been created.

If youu don't have the `brctl` command, use the `apt-get install bridge-utils to` to install the bridge tools.


## Step3
In the next step, we will creat a `veth` for connecting the `linux` bridge and the taget `container`.

The logical flow are
1. Get the bridge object from the `Bridge` we created before
2. Get the namespace of the container
3. Create a veth on the container and move the host-end veth to host ns.
4. Attach a host-end veth to linux bridge

This step is more complicate then previous steps. since we will handle the `network namespace` here.
Fortunately, the `CNI` project has provided convenience function to handle the veth and it can cover the (3) action itom above.

First, we use the `netlink.LinkByName` method to lookup the `netlink` object.

```go=
    l, err := netlink.LinkByName(sb.BridgeName)
    if err != nil {
        return fmt.Errorf("could not lookup %q: %v", sb.BridgeName, err)
    }
```

and the we need to make sure that object is `netlink.Bridge`, so we do the type casting.
```go=
    newBr, ok := l.(*netlink.Bridge)
    if !ok {
        return fmt.Errorf("%q already exists but is not a bridge", sb.BridgeName)
    }
```

Second, since the `CmdArgs` already provide the `network namespace ` path of the container, we can use the method from the `ns` package to load the object of the network namespace.

import `"github.com/containernetworking/plugins/pkg/ns"`

```go=
	netns, err := ns.GetNS(args.Netns)
	if err != nil {
		return err
	}
```

For each `NetNS` object, it implement a function `Do` which take a function as its parameter and that function's parameter is the caller's network namespace.

The `do` function will switch the network namespace to `NetNS` object itself and call the function(parameter) and feed the original network namespace as parameter.

See the following example to learn more about `do` function.

```go=
var handler = func(hostNS ns.NetNS) error {
    hostVeth, containerVeth, err := ip.SetupVeth(args.IfName, 1500, hostNS)
}

if err := netns.Do(handler); err != nil {
return err
}
```

First , we create a function handler which calls the `ip.SetpuVeth` to create a veth pair on caller's network namespace and move one side of veth pair to its third parameter(`hostNS`)

When we call the `netns.Do(handler)`, it will call the function `handler` in `netns's` network namepsace and pass the caller's network namespace to the `function handler`.
Which will result in that there will be a veth pair between the host's network namespace and `netns's` netowkr namespace.




In order to store the information about that veth pair, we can use the `current.Interface{}` object to store the data.

First, we need to import the library
```go=
import "github.com/containernetworking/cni/pkg/types/current"
```

and then create a variable represent to host side network interface in the function handler.

```go=
hostIface := &current.Interface{}
var handler = func(hostNs ns.Netns) error {
    hostVeth, _, err := ip.SetupVeth(args.IfName, 1500, hostNS)
    if err != nil {
        return err
    }

    hostIface.Name = hostVeth.Name
    return nil
}
```

Now, we can get the interface name of veth pair in the host side by `hostIface.Name` and then we will attach that link to the `Linux Bridge` we created before.

1. Get the link object from the interface name by function call `netlink.LinkByName`
2. Connect the link to bridge by function call `netlink.LinkSetMaster`

```go=
hostVeth, err := netlink.LinkByName(hostIface.Name)
if err != nil {
    return err
}

if err := netlink.LinkSetMaster(hostVeth, newBr); err != nil {
    return err
}

```

There is one important thing we need to care is the OS thread. since we will switch the `netns` to handle the namespace things.
We must make sure the OS won't switch the thread during the `namespace` operations.

Use the function `runtime.LockOSThread()` in the golang predefined function `init()`.

```go=
func init() {
        // this ensures that main runs only on main thread (thread group leader).
        // since namespace ops (unshare, setns) are done for a single thread, we
        // must ensure that the goroutine does not jump from OS thread to thread
        runtime.LockOSThread()
}```

See the whole example program in https://github.com/hwchiu/CNI_Tutorial_2018/tree/master/tutorial/step3 and you can directly run
the `run.sh` in your linux machine to see the following output.
```shell=
Ready to call the step3 example
{test 192.0.2.1/24}
The CNI has been called, see the following results
The bridge and the veth has been attatch to
bridge name     bridge id               STP enabled     interfaces
test            8000.aa6e12faa09b       no              vethff65a064
The interface in the netns
eth10     Link encap:Ethernet  HWaddr 7e:23:e2:e5:8f:c4
          inet6 addr: fe80::7c23:e2ff:fee5:8fc4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:90 (90.0 B)  TX bytes:90 (90.0 B)

lo        Link encap:Local Loopback
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

We have successfully create a linux bridge and connect to the other network namespace via the veth pair and the interface in that namepsace is `eth10` which has been defiend in the config file.


## Step4
In this step, we will setup the IP address into the target network namespace.
To make the problem easy, we had set the target IP address in the config and we can get via the `sp.IP`

```go=
type SimpleBridge struct {
        BridgeName string `json:"bridgeName"`
        IP         string `json:"ip"`
}
```

The function we used to assign the IP address is `netlink.AddrAdd`
So the workflow is
1. Generate a IP object from the config.
2. Call the `nelink.AddrAdd` in the target network namespace.

The parameter of `netlink.AddrAdd` is `netlink.Addr` and see its structure below.
```go=
type Addr struct {
        *net.IPNet
        Label       string
        Flags       int
        Scope       int
        Peer        *net.IPNet
        Broadcast   net.IP
        PreferedLft int
        ValidLft    int
}
```
We can use the `net` package provided by official golang to generate the `net.IPNet` type and its a CIDR form (IP address and the Mask).

Since the IP address in our config is a string`192.0.2.15/24`,
we use the `net.ParseCIDR` to parse the string and return a pointer of `net.IPNet`

So, modify the previous handler to assign the IP address when we create a veth.

Since the `net.IPNet` object get from the `net.ParseCIDR` is the `subnet`  not a `real IP` addrees, we should reassign the `IP` address to its IP field again.

```go=
var handler = func(hostNS ns.NetNS) error {
    hostVeth, containerVeth, err := ip.SetupVeth(args.IfName, 1500, hostNS)
    if err != nil {
        return err
    }
    hostIface.Name = hostVeth.Name

    ipv4Addr, ipv4Net, err := net.ParseCIDR(sb.IP)
    if err != nil {
        return err
    }

    link, err := netlink.LinkByName(containerVeth.Name)
    if err != nil {
        return err
    }

    ipv4Net.IP = ipv4Addr

    addr := &netlink.Addr{IPNet: ipv4Net, Label: ""}
    if err = netlink.AddrAdd(link, addr); err != nil {
        return err
    }
    return nil
}

```

See the whole example program in https://github.com/hwchiu/CNI_Tutorial_2018/tree/master/tutorial/step4 and you can directly run
the `run.sh` in your linux machine to see the following output.

```shell
Ready to call the step4 example
{test 192.0.2.15/24}
The CNI has been called, see the following results
The bridge and the veth has been attatch to
bridge name     bridge id               STP enabled     interfaces
test            8000.a6f55b2927c0       no              vethd611bb3b
The interface in the netns
eth10     Link encap:Ethernet  HWaddr aa:a0:96:45:65:c5
          inet addr:192.0.2.15  Bcast:192.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a8a0:96ff:fe45:65c5/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:168 (168.0 B)  TX bytes:90 (90.0 B)

lo        Link encap:Local Loopback
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

And you can see we have already set the IP address to the interface `eth10`.
You can use the following command to mamually set the IP address to the linux bridge and use the `ping` command to check the network connectiviy between the host and the target network namespace.
```shell
sudo ifconfig test 192.0.2.1
sudo ip netns exec ns1 ping 192.0.2.1
```


# Summary
In this tutorial, we have implemented a simple Linux Bridge CNI (only Add function) in golang.

We create the linux bridge and use the veth to connect the linux bridge with the target netowrk namespace.
Besides, we also fethc the information we want from the pre-defined config file which means we can more flexible to change the behavior of your own CNI implementation.

To make the problem simple, we don't use any complicated method to acquire a unique address from the config but you can desing you own algorithm to do that.
If you want to learn more about the IP related operations, you can go to the [host-local](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/host-local) to learn more.
