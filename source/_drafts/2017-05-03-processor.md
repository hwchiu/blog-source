---
layout: post
title: 'Processor'
date: 2017-05-03 02:50
comments: true
categories: 
---
Ceph Network - Processor
========================

### bind (const bind_addr, bound_addr)
- bind_addr:期望 bind 的 address
- bound_addr: 實際上 bound 的 addr

### start
- start thread
- 創造一個 accept_event 到 worker 的 eventcenter 內
- 此 accept_event 則是負責 listen 的功能

### accept
- call add_accept (get the worker from stack).
- Create AsyncConnectionRef to hold the connection.
### callback
- listen
### stop
- call event center to delete the listen event
- stop the listen socket.



###### tags: `ceph` `msg` `async`
