---
layout: post
title: How to enable Ceph with RDMA
date: '2017-05-03 02:47'
comments: true
tags:
  - Network
  - Ceph
  - RDMA
  - ScaleOut
abbrlink: 46014
---
Introduction
------------
- RDMA (Remote Direct Memory Access) is a mechanism which allow the host to accessing(read, write) memory on a remote host without interrupting CPU.
- The advantage of RDMA
	1. Zero-copy
  2. Kernel bypass
  3. No CPU involvement`
- With RDMA, our data can transfer without the involvement of the linux kernel network stack and provide hight performance, low latency, low CPU consumption.
- This article focus on how to enable the ceph with RDMA, including how to install ceph and enable the RDMA function.

<!--more-->


Install
-------
- I introduce two ways to install the ceph with RDMA, one is use widly used tool `ceph-deploy` and the other is manually build the ceph.

###ceph-deploy####
- If you use the `ceph-deploy` to install the ceph, you must make sure the source package  you installed is configure with `-DWITH_RDMA=ON`.
- You can use the argument **--dev** and **-dev--commit** to select the source packet form the official ceph build phase.
	- you can find those avaliabe repos in the [ceph site](https://shaman.ceph.com/repos/ceph/)
  - choose the one you want to install and clink it into the next page, you will see something like this **Repos ceph > wip-jd-testing > da2c3dabdad80c01ec3d3258b51640cc0a93e842 > default**
  - **wip-jd-testing** is for **--dev** and **da2c3...** is for **--dev-commit**.
  - use the following command to install the ceph from above repos.
```
  ceph-deploy install --dev=wip-jd-testing --dev-commit=da2c3dabdad80c01ec3d3258b51640cc0a93e842
```


###Manually build####
- Refer to followings step to build the ceph with RDMA.
```
cd ceph
./install-deps.sh
./do_cmake.sh -DWITH_RDMA=ON
cd build
time make -j54
sudo make install
```
- You can add any other options in command **do_cmake.sh**
- And than you should install the ceph to you environment and set up the monitor/osd by yourself.


Enable RDMA
-----------
- Before we enable the RDMA, there're something we need prepare for, including the RDMA environment, systemd config (if you need) and the ceph.conf

- Before we enable the RDMA, we must setup the RDMA environment, you should install the NIC driver and validate RDMA functionalities


#### RDMA environment
- I use the mellanox **ConnectX-3 Pro** in my environment and you can refer to [HowTo Enable, Verify and Troubleshoot RDMA](https://community.mellanox.com/docs/DOC-2086)
- Use rdma tools to make sure your RDMA work well.

#### Systemd config
- If you want to use the **systemd** to manage the ceph daemons, you should modify the systemd config to make it support RDMA because of the default config will fail for some access permission problem.
	- You can wait the official [PR](https://github.com/ceph/ceph/pull/14107/files) and use the next version.
  - Refer to this [PR](https://github.com/ceph/ceph/pull/13305) to modfiy the systemd config by yourself, and you can use `systemctl` reload the systemd config if you need.

#### ceph.conf
- Modify the **ms_type** to **async+rdma**, which tell the ceph use the **AsyncMessenger + RDMA** as your message type.
- You can use **ms_cluster_type** and **ms_public_type** to indicate the message type for your cluster network or public network.
- Use the command `ibdev2netdev` to get your device name and use it for **ms_async_rdma_device_name**
- If your want to use the port 2 in your NIC for RDMA, set the **ms_async_rdma_port_num** to 2.
- You can also use **ms_async_rdma_buffer_size**, **ms_async_rdma_send_buffers** and **ms_async_rdma_receive_buffers** to set the memory you want to allocate for RDMA.
	- **ms_async_rdma_send_buffers** and **ms_async_rdma_receive_buffers** are how many work requestes for RDMA send/receive queue respectively.
  - ms_async_rdma_buffer_size is the size os a single registered buffer.
  - the total memory we allocate for each application is ms_async_rdma_buffer_size * (ms_async_rdma_send_buffers + ms_async_rdma_receive_buffers) and you can refer to [here](https://github.com/ceph/ceph/pull/13510) to know more about it.
  
Example ceph.conf
```
[global]
...
ms_type=async+rdma
ms_async_rdma_device_name=mlx4_0
ms_async_rdma_send_buffers=1024
ms_async_rdma_receive_buffers=1024
...
```

Update the ceph.conf for each node and restart all daemons, after that, the ceph cluster will use the RDMA for all public/cluster network.
If you want ot make sure the RDMA works, you can use the following method to dump the RDMA packet and use the wireshark to open it.
```
1. echo "options mlx4_core log_num_mgm_entry_size=-1" || sudo -a tee /etc/modprobe.d/mlx4.conf
2. sudo  /etc/init.d/openibd restart 
3. ibdump
```

Reference
---------
-  [Mellanox-HowTo Configure Ceph RDMA](https://community.mellanox.com/docs/DOC-2693)
