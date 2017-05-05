---
layout: post
title: 'RDMAServerSocketImpl'
date: 2017-05-03 02:50
comments: true
categories: 
---
RDMA - RDMAServerSocketImpl
===========================
- **RDMAServerSocketImpl** 繼承自 **ServerSocketImpl**，重新實做了下列 function
    - accept
    - abort_accept
    - fd (回傳 server 用的 fd)
    - listen
    - get_fd (回傳 server 用的 fd)

- 待補其用途

#### listen
- 透過 **net_handler** 在本地創建一個 tcp socket
```c++
0029 int RDMAServerSocketImpl::listen(entity_addr_t &sa, const SocketOptions &opt)
0030 {
0031   int rc = 0;
0032   server_setup_socket = net.create_socket(sa.get_family(), true);
```
- 設定 socket options
    - non-blocking
    - nodely
    - rcbuf_size
    - close_on_exec
- bind 該 tcp socket
- list 該 tcp socket

#### accept
- 透過 `accept` 去接受對面的連線
- 設定 socket options
    - non-blocking
    - close_on_exec
    - nodelay
    - rcbuf_size
- 叫起一個 **RDMAConnectedSocketImpl** 的物件，
    - 將當前 accept 到的 fd 給上述物件
- 真正提供外面使用的介面是由 ConnectedSocket 來負責的，其裡面會包含 ConnectedSocketImpl，
    - ConnectedSocketImpl 實做了 read/send 等功能
- 結構來看則是
    - ConnectedSocket 包含了 ConnectedSocketImple (以 unique_ptr的形式)
- 所以在這邊先將 **RDMAConnectedSocketImpl** 給包成一個 **unique_ptr**
- 接者在創立一個 ConnectedSocket 並且將 **RDMAConnectedSocketImpl** 當作參數傳入
``` c++
0101   RDMAConnectedSocketImpl* server;
0102   //Worker* w = dispatcher->get_stack()->get_worker();
0103   server = new RDMAConnectedSocketImpl(cct, infiniband, dispatcher, dynamic_cast<RDMAWorker*>(w));
0104   server->set_accept_fd(sd);
0105   ldout(cct, 20) << __func__ << " accepted a new QP, tcp_fd: " << sd << dendl;
0106   std::unique_ptr<RDMAConnectedSocketImpl> csi(server);
0107   *sock = ConnectedSocket(std::move(csi));
0108   if (out)
0109     out->set_sockaddr((sockaddr*)&ss);
```

#### abort_accept
- 若 **server_setup_socket** 存在，則透過 `close` 將其關閉


###### tags: `ceph` `rdma` `async` `msg`