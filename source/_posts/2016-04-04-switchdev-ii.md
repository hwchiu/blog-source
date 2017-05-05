---
layout: post
title: 'Switchdev II'
date: 2016-04-04 07:01
comments: true
categories: 
tags:
	- System
	- Kernel
	- Switchdev
	- Network
---
Introduction
------------  
  此篇文章用來說明在當前 kernel 中, switchdev 相關的檔案有哪些，哪些是 switchdev 的核心，哪些是與原先的 linux kernel 整合，同時簡述一下各整合的用途為何。

<!--more-->


Architecture
------------
  switchdev 在 kernel 中的檔案架構如下
##Header##
####Linux Netowrk Function Integration####
- include/net/dsa.h
	- 這個是 **Distributed Switch Architecture**, 於 2015 年此 commit(b73adef) 將 switchdev 給整合進來
  - 根據 2008 的第一筆 commit log 來看, DSA是用來控制 hardware switch chips 的協定，不過大部分的功能都是在 2014 年後才慢慢實作，目前還無法確認此協定能夠做什麼
  
    >Distributed Switch Architecture is a protocol for managing hardware
    >switch chips.  It consists of a set of MII management registers and
    >commands to configure the switch, and an ethernet header format to
    >signal which of the ports of the switch a packet was received from
    >or is intended to be sent to. 
- include/inux/netdevice.h
  - net_device 用來代表整個 kernel 中所有的網路裝置，包含了常見的 network interface.
  - 主要針對 net_device 這個結構進行擴充，加上與 swticdev 的整合。  
  	 1. 加上 switchdev_ops 來提供相關的 operation
     2. 加上 offload_fwd_mark 來避免已經被 offload 的 packet 再次被 forward.
  
```
const struct switchdev_ops *switchdev_ops;  
u32                     offload_fwd_mark;
```
####SwtichDev Implemnetation####
- include/net/switchdev.h
	- switchdev.h 包含了所有的 struct, function, 要瞭解 switchdev 的核心就必須要看此檔案
  
##implementation##
####Linux Netowrk Function Integration####
  - net/8021q/vlan_dev.c
  	- 若底下的 bridge port 是個 vlan interface 的話，為了要能夠取得其 static FDB 以及 port 相關的狀態，在 net_device_ops 中把相關的 operation handler 都設定為 switchdev 的 function.
  - net/bridge/br.c
  - net/bridge/br_fdb.c
  - net/bridge/br_if.c
  - net/bridge/br_mdb.c
  - net/bridge/br_stp.c
  - net/bridge/br_stp_if.c
  - net/bridge/br_vlan.c
  	- 以上所有 bridge 相關的改動都是要將 hardware switch 的 l2 offload 與 linux kernel 給整合，包含了 STP/FDB/vlan/MDB 的變動。
    - 當底下 hardware switch 有任何變動時，都必須要主動通知 kernel 內的 bridge function 來處理。
  - net/core/net-sysfs.c
  	- export 一個新的 interface **/sys/class/net/<iface>/phys_switch_id**, 可用來知道 **iface** 此 port 所屬的 hardware switch ID
  - net/core/rtnetlink.c
  	- 新增一種 rtnl type **IFLA_PHYS_SWITCH_ID**, 可用來獲得特定 netdevice 所屬的 switch id。
    - 在 **iproute2** 也加入了此 type 的支援，意味者 user space 的 tool 也一併支援此功能了。
  - net/dsa/slave.c
  	- For DSA 使用，不熟所以忽略
  - net/ipv4/fib_trie.c
  	- 將 ipv4 的 FIB forwarding 與 hardware switch 整合，當 kernel 內關於 FIB 有任何更動時(ADD/DEL/MOD)時，要主動通知 hardware switch，將該 flow 加入到 ipv4 offload rules 中。
    - 並非所有的 FIB 都會通知底下，目前的規範是
    
```
/* Don't offload route if using custom ip rules or if
* IPv4 FIB offloading has been disabled completely.
*/
```
  
####SwtichDev Implementation####
- net/switchdev/switchdev.c
	- switchdev 的實作都在這邊，包含了與 hardware switch 以及 kernel 內相關 function 的互動。
  
  
###Switch Driver Implementation###
- drivers/net/ethernet/rocker/
- drivers/net/ethernet/mellanox/mlxsw/

目前 kernel 只有兩個真正的實作而已，而 rocker 算是作者開發 switchdev 中的共同產物，所以 mellanox 應該算是第一個進入的廠商。

