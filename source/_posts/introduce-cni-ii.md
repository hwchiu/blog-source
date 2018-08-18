---
title: '[CNI] Container Network Interface Introduction'
keywords: 'Network,Linux,Ubuntu,Docker,Kernel,CNI'
tags:
  - CNI
  - Network
  - Docker
  - Linux
  - Ubuntu
  - Kubernetes
abbrlink: 26571
date: 2018-04-08 05:16:01
description:
---

I have write three posts about the CNI and you can found the other posts below
[[CNI] Bridge Network In Docker](https://www.hwchiu.com/introduce-cni-i.html)
[[CNI] Write A CNI Plugin By Golang](https://www.hwchiu.com/introduce-cni-iii.html)



In this post, I will try to introduce the concept of Container Network Interface (CNI), including why we need this, how it works and what does it do.

If you have not familiar with what is `linux network namespace` and how `docker` handles the network for its containers.
You should read the [[CNI] Bridge Network In Docker](https://www.hwchiu.com/introduce-cni-i.html#more) to learn those concepts and that will be helpful for this tutorial.

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


Kubernetes
==========

Now, We will see how kubernetes use CNI to create a network function for Pods.

## Configuration
In order to use the CNI, we need to config the `kubelet` to use the `CNI` method.
There're three argurments we need to take care.
1. cni-bin-dir: the directory of CNI binaries.
2. cni-conf-dir: the directory of CNI config files, common CNI(flannel/calico..etc) will install its config into here.
3. network-plugin: the type of network-plugin for Pods.


In my kubernetes cluster (installed by kubeadm)
```shell=
vortex-dev:10:06:59 [~]vagrant
$ps axuw | grep cni
root      1864  4.9  2.1 569172 110108 ?       Ssl  15:18   3:06 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d --network-plugin=cni
```
You can see the arguments `--cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d --network-plugin=cni` of the kubelet.

Now, Let we see the files under `cni-bin-dir` and `cni-conf-dir`.

The `cni-bin-dir` contains all the CNI binary file and those files can be programmed by any language, just follow the CNI interface. 
```shell=
vortex-dev:04:21:29 [~]vagrant
$ls /opt/cni/bin/
bridge  dhcp  flannel  host-local  ipvlan  loopback  macvlan  portmap  ptp  rainier  sample  tuning  vlan
```


In the `cni-conf-dir`, we should put the CNI config here and `kubernetes` will use the config for your Pod.
In my `kubernetes` cluster, I had installed the flannel CNI in it and the flannel will install its config here.
```shell=
vortex-dev:05:11:30 [~]vagrant
$ls /etc/cni/net.d/
10-flannel.conf
vortex-dev:05:11:34 [~]vagrant
$cat /etc/cni/net.d/10-flannel.conf
{
  "name": "cbr0",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true
  }
}
```

## How To Use it.
When `kubelet` receives a request to create a Pod in the node.
First, it will search the `cni-conf-dir` in the alphabet order and inspect it.

Take the `10-falnnel.conf` as example. when the `kubelet` knows the `type` is `flannel`, it will try to call the `flannel` in the `cni-bin-dir` and that's `/opt/cni/bin/flannel`.

```shell=
vortex-dev:05:11:34 [~]vagrant
$cat /etc/cni/net.d/10-flannel.conf
{
  "name": "cbr0",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true
  }
}
```

## Pause Container.
Before `kuberlet` create the Pod, it will create a `pause` conatiner first.
And follows the CNI steps to setup the network fot that `pause` container.(Assueme we use the network-plugin=cni)

Now, The pause container is running and has the network connectivity.
The `kubelet` will create containers which is be described in the yaml file and attach those container to that pause container (in the docker command, we can use the --net=$containerID to do the same thing).

By those procedure, we can maks sure all containers share the same network stack and any container crash won't destory the network stack since the network stack is hold sy the `pause container`.

Combine the pause container and user containers, it's called `Pod`.
And you can try to use the `docker ps` in your `kubernetes` node to see how many pause container in there.

```shell=    
vortex-dev:05:19:30 [~]vagrant
$sudo docker ps -a | grep pause
8838b9614a30        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
                   k8s_POD_nfs-provisioner-5b75397b4807c54ad4fe92e2-6954c749cc-cn5jh_vortex_9f2f692c-a130-11e8-9450-02ddf6cab53d_0
0a232459f786        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
                   k8s_POD_vortex-server-58895cd7c6-xvd8g_vortex_7d88347b-9f9a-11e8-8719-02ddf6cab53d_8
b0ca4ca2405d        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
                   k8s_POD_kube-state-metrics-7d7d7b6bbc-fsf7b_vortex_7d83db65-9f9a-11e8-8719-02ddf6cab53d_7
63a1f3b8a35f        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
                   k8s_POD_coredns-78fcdf6894-s8ts5_kube-system_c9ef514c-9a23-11e8-9c21-02ddf6cab53d_9
310b7a6daa54        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
                   k8s_POD_cadvisor-zk8bk_vortex_7d726ff5-9f9a-11e8-8719-02ddf6cab53d_3
3f0141a5a9b6        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
                   k8s_POD_network-controller-server-tcp-nnvgk_vortex_7d648d43-9f9a-11e8-8719-02ddf6cab53d_2
9cedcb482e69        k8s.gcr.io/pause:3.1                              "/pause"                 7 hours ago         Up 7 hours
```

Summary
=======
The `Container Network Interface` CNI made the network-service developer more easy to develop their own network plguin. They don't need to write duplicate code for different system/orchestrator.
Just write once and run everywhere.

And the CNI consists of a specification and many userful libraries for developers. The CNI only care the `ADD` and `DELETE` events. the CNI plugin shoould make sure the container with network connectivity when the `ADD` event has been triggered and remove all allocted resources when the `DELETE` event has been triggered.

In the next tutorial, I will show how to write a simple bridge CNI plugin in golang.
