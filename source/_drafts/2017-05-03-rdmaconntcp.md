---
layout: post
title: 'RDMAConnTCP'
date: 2017-05-03 02:49
comments: true
categories: 
---
RDMAConnTCP
===========

#### Constructor
- 取得當前 RDMA 使用的 device name
	- ceph.conf 中的設定值`ms_async_rdma_device_name`
- 取得當前 RDMA 使用的 port num
	- ceph.conf 中的設定值`ms_async_rdma_port_num`
- 透過 **Device** 去初始化該 IB device
- 根據該 `device + port` 去創造對應的 **QueuePair**
``` c++
QueuePair *qp = socket->create_queue_pair(ibdev, ibport);
```
- 初始化 msg 相關的資訊
``` c++
0069   my_msg.qpn = socket->local_qpn;
0070   my_msg.psn = qp->get_initial_psn();
0071   my_msg.lid = ibdev->get_lid(ibport);
0072   my_msg.peer_qpn = 0;
0073   my_msg.gid = ibdev->get_gid(ibport);
```
- 最後註冊該 qp
``` c++
0074   socket->register_qp(qp);
```
- 如果有傳入 info  資訊的話，根據資訊內容創造對應的 tcp socket
	- server side 才會有 info 資訊。
	- 供 control plane 使用


###### tags: `ceph` `rdma` `async` `msg`