---
layout: post
title: 'AsyncMessenger'
date: 2017-05-03 02:51
comments: true
categories: 
---
Ceph Network - AsyncMessenger
=============================
### Construct
- Create networkstack (according the ms_type from ceph.conf)
- start the networkstack
- Create local worker with a AsyncConnection.
- init the local_connection
- init all processors, (if support local_listen_table, the number of processor will bigger than 1).\
- create read_dead_connection handler
``` c++
// backend need to override this method if backend doesn't support shared
// listen table.
// For example, posix backend has in kernel global listen table. If one
// thread bind a port, other threads also aware this.
// But for dpdk backend, we maintain listen table in each thread. So we
// need to let each thread do binding port.
virtual bool support_local_listen_table() const { return false; }
```

### Deconstruct
- delete read_dead_connection handler
- cloas all local connection
- delete all processors

### Ready
- NetworkStack start again
    - Create worker based on valud of num_workers
- Start all processors
    - create a listen event and pass it to the eventcenter.
    - the listen event will create a AsyncConnection and call its accpet when the listen fd receive a new coming connection.
    - add the connectiong into the global variable (accepting_conns)
- Start dispatch_queue

### Shutdown
- Stop all processros
- call mark_down_all
    - down all connections.
- drain the networkstack, make sure its task has completed.
    - See more about drain concept, refet to [here](http://stackoverflow.com/questions/34304286/asioio-service-and-thread-group-lifecycle-issue)

### Bind
- all of processor do bind function
``` c++
0336   for (auto &&p : processors) {
0337     int r = p->bind(bind_addr, avoid_ports, &bound_addr);
0338     if (r) {
0339       // Note: this is related to local tcp listen table problem.
0340       // Posix(default kernel implementation) backend shares listen table
0341       // in the kernel, so all threads can use the same listen table naturally
0342       // and only one thread need to bind. But other backends(like dpdk) uses local
0343       // listen table, we need to bind/listen tcp port for each worker. So if the
0344       // first worker failed to bind, it could be think the normal error then handle
0345       // it, like port is used case. But if the first worker successfully to bind
0346       // but the second worker failed, it's not expected and we need to assert
0347       // here
0348       assert(i == 0);
0349       return r;
0350     }
0351     ++i;
0352   }
```
- call _finish_bind 
    - init local connection
### Rebind
- Try to rebind, do the same thine as **`bind`**
- After the binding, start all processors
### Client_bind
- only record the client's address.
### Start
- init variable
    - started
    - stopped
    - etc
- init_local_connection
### Wait
- dispatch_queue.shutdown
- dispatch queue wait
- dispatch qeuue drop local
- shutdown all connections
### Add_accept
- New a AsyncConnection and call its function to aceept.
``` c++
1863 void AsyncConnection::accept(ConnectedSocket socket, entity_addr_t &addr)
1864 {
1865   ldout(async_msgr->cct, 10) << __func__ << " sd=" << socket.fd() << dendl;
1866   assert(socket.fd() >= 0);
1867 
1868   std::lock_guard<std::mutex> l(lock);
1869   cs = std::move(socket);
1870   socket_addr = addr;
1871   state = STATE_ACCEPTING;
1872   // rescheduler connection in order to avoid lock dep
1873   center->dispatch_event_external(read_handler);
1874 }
```
- add the connection in to global variable (accepting_conns)

### Create_connect
- get the worker from networkstack
- new a AsyncConnection
- try to connect to target
- add the connection into a global variable (conns)
- add the perf counter for worker

### Get_connection
- if the input address is the same as myself addr, return the local connection.
- look up the dest addr from conns
- if exist return the connection, else create a new connection via `create_connect` then return.

### Get_loopback_connection
- return local connection

### _send_message
- decrease the usage counter of the message (`m->put()`)
- find the connection based on the destination address.
- excute t he submit_message to send the message

### submit_message
- dump the message conten if the ms_dump_on_send is true
- send the message if the connection exist
- send the message if the connection is local connection
- is the connection doesn't exist
    - if the policy type is server, it doesn't allow reconnect and we discard this packet.
    - Otherwise, create a new connection and send the message

### set_addr_unknowns
- if the record of the current addres is blank, use the input address as my own address.

### shutdown_connections
- clear all accepting_conns 
- clear all conns
- clear all dead_conns

### mark_down
- stop the connection

### get_proto_version
- get the peer's protocol version
``` c++
#define CEPH_OSDC_PROTOCOL   24 /* server/client */
#define CEPH_MDSC_PROTOCOL   32 /* server/client */
#define CEPH_MONC_PROTOCOL   15 /* server/client */
```

### learned_addr
- use the input address as myself address.
- init local connection

### read_dead
- clear all connection within deleted_conns
- also remove that connection from accepting_conns/conns

###### tags: `ceph` `async` `msg`
