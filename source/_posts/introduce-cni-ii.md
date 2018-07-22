---
title: Introduction to Container Network Interface(II)
keywords: 'Network,Linux,Ubuntu,Docker,Kernel,CNI'
date: 2018-04-08 05:16:01
tags:
	- CNI
	- Network
	- Docker
	- Linux
	- Ubuntu
	- Kubernetes
description:
---

In this post, I will try to introduce the concept of Container Network Interface (CNI), including why we need this, how it works and what does it do.

If you have not familiar with what is `linux network namespace` and how `docker` handles the network for its containers.
You should read the [Introduction to Container Network Interface(I)](http://hwchiu.com/introduce-cni-i.html#more) to learn those concepts and that will be helpful for this tutorial.

<!--more-->

Introduction
============
## Why We Need CNI
In the previous post, we have learn the procedure of the basic bridge network in the docker.
- Create a Linux Bridge
- Create a Network Namespace
- Create a Veth Pair
- Connect the bridge and network namespace with veth pair
- Setup the IP address to the network namespace
- Setup the iptalbes rules for exporting the services (optional)

However, That's the `bridge network` and it only provide the layer2 forwarding. For some use cases, it's not enough.
More and more requirement, such as layer3 routing, overlay network, high performance
, openvswitch and so on.


From the docker point of view, it's impossible to implement and maintain all above requirements by them.

The better solution is to open its interface and make everyone can write its own network service and that's how `docker network` works.

So, there're so many plugins for the `docker network` now and every can choose what kind of the network they want.

Unfortunately, docker isn't the only container technical, there're otehr competitors, such as `rkt`, `lxc`.
Besides, more and more `container cluster orchestration`, `docker swam`, `mesos`, `kubernetes` and so on.

Take a `bridge network` as an example, do we need to implement the `bridge network` for all container orchestration/solutions? do we need to write many duplicate code because of the not-unified interface between each orchestrator?

That's why we need the `Container Network Interface(CNI)`, The `Container Network Interface(CNI)` is a `Cloud Native Computing Foundation` projects, we can see more information [here](https://github.com/containernetworking/cni).

With the `CNI`, we have a unified interface for network services and we should only implement our network plugin once, and it should works everywhere which support the `CNI`.

According to the official website's report. those `container runtimes` solutions all supports the `CNI`
-   [rkt - container engine](https://coreos.com/blog/rkt-cni-networking.html)
-   [Kubernetes - a system to simplify container operations](http://kubernetes.io/docs/admin/network-plugins/)
-   [OpenShift - Kubernetes with additional enterprise features](https://github.com/openshift/origin/blob/master/docs/openshift_networking_requirements.md)
-   [Cloud Foundry - a platform for cloud applications](https://github.com/cloudfoundry-incubator/cf-networking-release)
-   [Apache Mesos - a distributed systems kernel](https://github.com/apache/mesos/blob/master/docs/cni.md)
-   [Amazon ECS - a highly scalable, high performance container management service](https://aws.amazon.com/ecs/)


## How CNI works
`Container Network Interface` is a specifiction which defined what kind of the interface you should implement. 

In order to make it easy for developers to deveploe its own CNI plugin. the `Container Network Interface` project also provides many library for developing and all of it is based on the `golang` language.

You can find those two libraries below
[https://github.com/containernetworking/cni](https://github.com/containernetworking/cni)
[https://github.com/containernetworking/plugins](https://github.com/containernetworking/plugins)

## What does CNI do

In CNI specifiction, there're three method we need to implement for our own plugin.
- ADD
- DELETE
- VERSION

`ADD` will be invoked when the container has been created. The plugin should prepare resources and make sure that container with network connectivity.
`DEKETE` will be inboked when the container has been destroyed. The plugin should remove all allocated reousrces.
`VERSION` shows the version of this CNI plugin.


For each method, the CNI interface will pass the following information into your plugin
- ContainerID
- Netns
- IfName
- Args
- Path
- StdinData

I will explain those fields detaily in the next tutorial. In here, we just need to know for the CNI plugin, we sholud use those information `ContainerID`, `Network Namespace path` and `Interface Name` and `StdinData` to make the container with network connectivity.

Use the previous bridge-network as example. the `network namespace` will be created by the `orchestrator` and it will pass the path of that `network namespace` via the variable `netns` to CNI.
After we crete the `veth` pair and connect to the `network namespace`, we should set the interface name to `Ifname`.

For the IPAM (IP Adderss Management), we can get the information from the `StdinData` and calculate what IP address we should use in the CNI plugin.

In the next tutorial, I will show how to write a simple bridge CNI plugin in golang.

Summary
=======
The `Container Network Interface` CNI made the network-service developer more easy to develop their own network plguin. They don't need to write duplicate code for different system/orchestrator.
Just write once and run everywhere.

And the CNI consists of a specification and many userful libraries for developers. The CNI only care the `ADD` and `DELETE` events. the CNI plugin shoould make sure the container with network connectivity when the `ADD` event has been triggered and remove all allocted resources when the `DELETE` event has been triggered.

