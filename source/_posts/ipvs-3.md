---
title: IPvS 學習手冊(三)
tags:
  - IPVS
  - Network
  - Linux
  - Kernel
keywords: 'Kubernetes,Network,Linux,Ubuntu'
description: >-
  本文作為 IPVS 系列文第三篇， 會從 Kernel 作為出發點，探討一下 IPVS
  本身的模組概念，分享兩種不同的內建除錯方式，此外也會從原始碼的部分看一下 IPVS 初始化的過程做了哪些事情
abbrlink: 9470
date: 2020-04-06 00:03:29
---


# Preface
本篇文章作為系列文章的第三篇，該系列文希望能夠從概念到實作，從簡單到複雜來探討 IPVS (IP Virtual Server) 的概念，目前規劃的主題包含：
- [IPVS 的基本使用與概念](https://www.hwchiu.com/ipvs-1.html)
- [IPVS 與 Kubernetes 的整合](https://www.hwchiu.com/ipvs-2.html)
- [IPVS 除錯方式與基本 Kernel Module 概念](https://www.hwchiu.com/ipvs-3.html)

本文主要是從 Linux Kernel 出發，介紹一下對於 **IPVS** 這個模組的概念，同時也介紹了預設的兩種除錯方式。


# 環境
整篇文章都是基於 Linux Kernel 4.15 為基準去閱讀, 可以從 **[Github](https://github.com/torvalds/linux/tree/v4.15/net/netfilter/ipvs)** 或是 **[LXR](https://elixir.bootlin.com/linux/v4.15/source)** 來進行線上閱讀

本篇沒有環境建置，因為懶得打包整個 **建置 Linux Kernel** 的環境到 **Vagrant** 裡面。可能要等全部文章都寫完再補齊


# 正題

研究都要有個起點，整個 Linux Kernel 過於龐大，實在是不可能全部閱讀。
今天要研究的是 **IPVS** 相關的功能，我們就從其[資料夾](https://github.com/torvalds/linux/tree/v4.15/net/netfilter/ipvs)內開始看起。

根據先前的實驗，我們使用 **IPVS** 之前必須要先安裝對應的 **kernel module**，每個 **kernel module** 都會有相關的初始化函式可以使用。
所以我們就先從 **[Makefile](https://github.com/torvalds/linux/blob/v4.15/net/netfilter/ipvs/Makefile)** 看起來，看一下到底這些 **kernel module** 實際上包含了哪些檔案 


```makefile
# SPDX-License-Identifier: GPL-2.0
#
# Makefile for the IPVS modules on top of IPv4.
#

# IPVS transport protocol load balancing support
ip_vs_proto-objs-y :=
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_TCP) += ip_vs_proto_tcp.o
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_UDP) += ip_vs_proto_udp.o
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_AH_ESP) += ip_vs_proto_ah_esp.o
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_SCTP) += ip_vs_proto_sctp.o

ip_vs-extra_objs-y :=
ip_vs-extra_objs-$(CONFIG_IP_VS_NFCT) += ip_vs_nfct.o

ip_vs-objs :=	ip_vs_conn.o ip_vs_core.o ip_vs_ctl.o ip_vs_sched.o	   \
		ip_vs_xmit.o ip_vs_app.o ip_vs_sync.o	   		   \
		ip_vs_est.o ip_vs_proto.o ip_vs_pe.o			   \
		$(ip_vs_proto-objs-y) $(ip_vs-extra_objs-y)


# IPVS core
obj-$(CONFIG_IP_VS) += ip_vs.o

# IPVS schedulers
obj-$(CONFIG_IP_VS_RR) += ip_vs_rr.o
obj-$(CONFIG_IP_VS_WRR) += ip_vs_wrr.o
obj-$(CONFIG_IP_VS_LC) += ip_vs_lc.o
obj-$(CONFIG_IP_VS_WLC) += ip_vs_wlc.o
obj-$(CONFIG_IP_VS_FO) += ip_vs_fo.o
obj-$(CONFIG_IP_VS_OVF) += ip_vs_ovf.o
obj-$(CONFIG_IP_VS_LBLC) += ip_vs_lblc.o
obj-$(CONFIG_IP_VS_LBLCR) += ip_vs_lblcr.o
obj-$(CONFIG_IP_VS_DH) += ip_vs_dh.o
obj-$(CONFIG_IP_VS_SH) += ip_vs_sh.o
obj-$(CONFIG_IP_VS_SED) += ip_vs_sed.o
obj-$(CONFIG_IP_VS_NQ) += ip_vs_nq.o

# IPVS application helpers
obj-$(CONFIG_IP_VS_FTP) += ip_vs_ftp.o

# IPVS connection template retrievers
obj-$(CONFIG_IP_VS_PE_SIP) += ip_vs_pe_sip.o
```

這個 Makefile 看起來會覺得好像什麼都沒有，最主要的原因是整體的運作邏輯都在 Kernel 其他檔案裡面，特別是 [Makefile.build](https://github.com/torvalds/linux/blob/v4.15/scripts/Makefile.build) ，這邊只是單純定義了每個 **Obj** 的狀況，看是要採取 **Build-in** 模式或是 **kernel module** 模式。

先複習一下之前安裝過程中所載入的 kernel module
```bash=
sudo modprobe -- ip_vs
sudo modprobe -- ip_vs_rr
sudo modprobe -- ip_vs_wrr
sudo modprobe -- ip_vs_sh
```

這邊幾個可以注意到的是
1. ipvs.ko 是由 ip_vs-objs 這一堆 objs 全部組合而成的。
2. 每個 **load balancing(scheduler)** 都是一個獨立的 kernel module，剛好對應到 **ip_vs_rr**, **ip_vs_wrr**...等，如果你要用不同的算法，那要確認對應的 module 都有被載入

有了這個方向，我們就從 **ip_cs-objs** 裡面的開始看起來，其中 **ip_vs_core** 這個名稱非常顯眼，看起來就是核心功能，因此決定之後從該檔案開始看起。

# Debug IPVS

開始往下閱讀 **IPVS** 原始碼之前，我們先來探討一下如何除錯以及如何學習相關的函式呼叫順序

不修改任何原始碼的情況下， **IPVS** 提供了兩種 **Debug** 的方式讓管理員去檢視相關的函式呼叫情況，這邊要注意的是，其提供的是 **函式呼叫情況**, 而不是一個能夠一一檢視封包的方式。就我個人的使用情況來說，如果本身對於 **IPVS** 的實作與 **Kernel** 不熟悉的人，就沒有辦法 100% 的利用這些除錯訊息。
這邊跟大家介紹這兩種除錯方式該如何開啟

## Dynamic Option
第一種方式是整個 **Kernel** 所提供的除錯方式, 稱之為 [dynamic-debug](https://www.kernel.org/doc/html/v4.19/admin-guide/dynamic-debug-howto.html)，所有 **kernel function** 都可以向 **kernel** 註冊一組輸出格式與條件，當今天管理員有需求時就可以動態的打開這些開關使得 **kernel** 開始輸出相關訊息。

系統上支援的 **dynamic debug** 都存放於 `/sys/kernel/debug/dynamic_debug/control` 這個路徑，透過 **cat** 的方式可以看到的內容都是由 **filename:line_number, [kernel_module name], topic, output_format** 這樣的格式，譬如下列範例
```bash=
cat /sys/kernel/debug/dynamic_debug/control
net/netfilter/xt_LOG.c:60 [xt_LOG]log_tg_check =_ "prefix is not null-terminated\012"
net/netfilter/xt_LOG.c:55 [xt_LOG]log_tg_check =_ "level %u >= 8\012"
net/bridge/netfilter/ebtables.c:2260 [ebtables]compat_do_replace =_ "tmp.entries_size %d, kern off %d, user off %d delta %d\012"
net/bridge/netfilter/ebtables.c:2129 [ebtables]size_entry_mwt =_ "change offset %d to %d\012"
net/bridge/netfilter/ebtables.c:1775 [ebtables]compat_calc_entry =_ "0x%08X -> 0x%08X\012"
net/netfilter/ipvs/ip_vs_proto.c:266 [ip_vs]ip_vs_tcpudp_debug_packet_v6 =_ "%s: %s %s\012"
net/netfilter/ipvs/ip_vs_proto.c:234 [ip_vs]ip_vs_tcpudp_debug_packet_v4 =_ "%s: %s %s\012"
```
前面代表的是每個檔案的路徑與行數，再來是對應的 **kernel module name** 以及標題，最後是其輸出的格式。

對於管理者來說，可以針對 **檔案名稱, 行數 或是 kernel module name 來進行開關，譬如我今天想要打開跟 **ipvs** 有關的所有輸出，就可以執行

```bash=
echo -n 'module ip_vs +p' > /sys/kernel/debug/dynamic_debug/control
```
這樣 **kernel** 就會知道把 **control** 裡面全部屬於 **ip_vs** 的訊息都輸出，譬如

```bash=
[ 3415.992622] ip_vs_tcpudp_debug_packet_v4:234: IPVS: ip_vs_in: packet continues traversal as normal: TCP 10.0.2.2:58199->10.0.2.15:22
```
對於想要細部觀察封包流向來說，也是不無小補，提供一些資訊觀察

## IP_VS_DEBUG

第二個則是由 **IPVS** 自己實作的除錯方式，這個除錯方式相關的功能則是在編譯期間根據參數 **IP_VS_DEBUG** 來定是否要一起編譯, 預設情況下這個功能是關閉的，這也意味各位如果都適用已經建置好的 **IPVS** 那基本上是沒有辦法打開這個功能的。 
為了打開這個功能，你必須要執行下列步驟
1. 準備對應你系統版本的 kernel source code
2. 加入相關的參數，重新建置 IP_VS 的 kernel module
3. 移除舊有的 kernel module 並安裝新的到系統內

如果你有興趣要嘗試看看的話，這邊可以顯示一下大概的修改過程 (假設你準備好相關的 kernel source code) 之後

1. 先複製系統目前使用的 kernel config (Ubutnu 為範例)
2. 根據該設定檔案重新加入 IP_VS_DEBUG 的參數
3. 重新建置 Kernel Module
4. 移除舊有的 Kernel Module, 並且加入新的

```bash=
cp /boot/config-`uname -r` .config
make menuconfig
sudo rmmod ip_vs_wlc
sudo rmmod ip_vs
sudo insmod net/netfilter/ipvs/ip_vs.ko
sudo insmod net/netfilter/ipvs/ip_vs_wlc.ko
make -j4 
```
其中第二部會彈出一個灰色畫面，可以透過 `f` 進行參數的搜尋，找到對應的位置，然後將其打該設定成 **Y** 即可，畫面如下
![](https://i.imgur.com/TuejssF.png)
![](https://i.imgur.com/VakjFDI.png)
圖中也可以看到 **IPVS** 滿滿的參數有哪些，其中標示為 <M> 都代表會建置成獨立的 **kernel module**，如果有興趣的也可以將全部變成 **Build-in** 的方式，這意味 **kernel** 本身就會包含這些功能，不需要額外在那邊 **insmod/rmmod**。


此外如果你本身已經有建置過 **kernel** 的話，可以直接輸入 **make modules** 單純編譯 **kernel modules** 相關即可

完成上述步驟之後就可以在系統上的位置發現多了一個路徑 **/proc/sys/net/ipv4/vs/debug_level**
![](https://i.imgur.com/f4yNxpd.png)

該路徑對應到 **kernel** 內則是一個整數，預設是 **0**, 今天如果要打開 **debug** 功能的話，就輸入數字到該路徑即可，譬如
```bash=
 echo "12" > /proc/sys/net/ipv4/vs/debug_level
```

一但該功能打開後，就會看到系統噴出滿滿的除錯訊息，譬如
```bash=
[391968.554844] IPVS: Enter: ip_vs_out, net/netfilter/ipvs/ip_vs_core.c line 1352
[391968.554846] IPVS: Enter: ip_vs_proto_name, net/netfilter/ipvs/ip_vs_core.c line 83
[391968.554847] IPVS: lookup/out TCP 172.17.8.111:37926->172.17.8.156:80 not hit
[391968.554848] IPVS: Enter: sysctl_nat_icmp_send, net/netfilter/ipvs/ip_vs_core.c line 671
[391968.554849] IPVS: Enter: ip_vs_local_request4, net/netfilter/ipvs/ip_vs_core.c line 2074
[391968.554850] IPVS: Enter: ip_vs_in, net/netfilter/ipvs/ip_vs_core.c line 1884
[391968.554851] IPVS: Enter: ip_vs_proto_name, net/netfilter/ipvs/ip_vs_core.c line 83
[391968.554852] IPVS: lookup/in TCP 172.17.8.111:37926->172.17.8.156:80 hit
[391968.554853] IPVS: Enter: is_new_conn, net/netfilter/ipvs/ip_vs_core.c line 1084
[391968.554854] IPVS: Enter: sysctl_expire_nodest_conn, net/netfilter/ipvs/ip_vs_core.c line 677
[391968.554854] IPVS: Enter: is_new_conn_expected, net/netfilter/ipvs/ip_vs_core.c line 1111
[391968.554855] IPVS: Enter: ip_vs_in_stats, net/netfilter/ipvs/ip_vs_core.c line 117
[391968.554856] IPVS: Enter: ip_vs_set_state, net/netfilter/ipvs/ip_vs_core.c line 209
[391968.554858] IPVS: Enter: ip_vs_nat_xmit, net/netfilter/ipvs/ip_vs_xmit.c line 756
[391968.554859] IPVS: Enter: __ip_vs_get_out_rt, net/netfilter/ipvs/ip_vs_xmit.c line 325
[391968.554860] IPVS: Enter: __ip_vs_dst_check, net/netfilter/ipvs/ip_vs_xmit.c line 98
[391968.554861] IPVS: Enter: crosses_local_route_boundary, net/netfilter/ipvs/ip_vs_xmit.c line 179
[391968.554862] IPVS: Enter: decrement_ttl, net/netfilter/ipvs/ip_vs_xmit.c line 269
[391968.554863] IPVS: Enter: ensure_mtu_is_adequate, net/netfilter/ipvs/ip_vs_xmit.c line 226
[391968.554865] IPVS: Enter: ip_vs_nat_send_or_cont, net/netfilter/ipvs/ip_vs_xmit.c line 624
[391968.554868] IPVS: Enter: ip_vs_nat_send_or_cont, net/netfilter/ipvs/ip_vs_xmit.c line 641
```



# Module Init
如上面所述， 我們將從 **ip_vs_core.c** 來探討整個 **ip_vs** 的初始化過程
以下的程式碼來自 [Linux 4.15 GitHub ip_vs_core.c](https://github.com/torvalds/linux/blob/v4.15/net/netfilter/ipvs/ip_vs_core.c)


每個 **Kernel Module** 都會從 **module_init** 這邊開始，傳入的參數都會是一個 **function**，當 Module 被載入後這個 function 就會被執行，也可以想成 **modprobe ip_vs** 呼叫後，這個 function 就會先被執行
```c=
module_init(ip_vs_init);
```

根據上述語法，可以觀察到 **ip_vs_init** 這個 **function**
```c=
static int __init ip_vs_init(void)
{
	int ret;

	ret = ip_vs_control_init();
	if (ret < 0) {
		pr_err("can't setup control.\n");
		goto exit;
	}

	ip_vs_protocol_init();

	ret = ip_vs_conn_init();
	if (ret < 0) {
		pr_err("can't setup connection table.\n");
		goto cleanup_protocol;
	}

	ret = register_pernet_subsys(&ipvs_core_ops);	/* Alloc ip_vs struct */
	if (ret < 0)
		goto cleanup_conn;

	ret = register_pernet_device(&ipvs_core_dev_ops);
	if (ret < 0)
		goto cleanup_sub;

	ret = ip_vs_register_nl_ioctl();
	if (ret < 0) {
		pr_err("can't register netlink/ioctl.\n");
		goto cleanup_dev;
	}

	pr_info("ipvs loaded.\n");

	return ret;

cleanup_dev:
	unregister_pernet_device(&ipvs_core_dev_ops);
cleanup_sub:
	unregister_pernet_subsys(&ipvs_core_ops);
cleanup_conn:
	ip_vs_conn_cleanup();
cleanup_protocol:
	ip_vs_protocol_cleanup();
	ip_vs_control_cleanup();
exit:
	return ret;
}
module_init(ip_vs_init);
module_exit(ip_vs_cleanup);
```

通常觀察一個 `kernel module` 就是先觀察 `init function`, 看看到底初始化哪些相關的資訊，其中上面呼叫的函示有
1. ip_vs_control_init
用來幫內部一些資料結構初始化，主要都是 HASH table 相關的資料結構
2. ip_vs_protocol_init
用來初始化支持的 `Protocol`，譬如 `TCP,UDP,SCTP` 等
根據前述的 **make menuconfig** 我們可以觀察到相關協定的支持也是可以透過參數去開關的。

4. ip_vs_conn_init
主要是用來初始化跟 `connection` 有關的資訊，`connection` 可以想成 `Client` 的請求與配置 `Real Server` 的關係表。 也是透過 `connection` 來確保相同連線的封包都會被轉發到相同的 `Real Server` 上
5. register_pernet_subsys/register_pernet_device
上述兩個函式非常有趣，他們的都是用來註冊 `pernet` 底下相關元件的函式
，這邊的 `pernet` 的意思就是 **每個 Network Namespace**。 等等我們再來仔細看看到底做了什麼
7. ip_vs_register_nl_ioctl
用來註冊基於 `netlink` 的 `function handler`，這也是我們透過 `ipvsadm` 這個工具用來操作整個 `kernel` 內部邏輯的入口。所有的操作指令都會透過 `netlink` 從 `userspace` 送到 `kernel space` 並且呼叫起對應的 `Function` 來處理。


## register_pernet_subsys/device
Kernel 對於 **network namespace** 的創建提供了兩個方法來註冊相關的 **handler**, 這兩個差異主要在於對背後資料結構操作的位置不同，詳細的可以參閱 [Kernel network namespace
](http://www.programmersought.com/article/907433913/), 這邊就單純針對 `register_pernet_subsys` 來研究。 

直接看一下該函式的註解，簡單來說就是註冊一組任何 **network namespace** 被刪除與創建時都會被呼叫的函式，而要特別注意的是當註冊的瞬間，也會對所有已經存在的 **network namespace** 呼叫一次。
```c=
 *      register_pernet_subsys - register a network namespace subsystem
 *      @ops:  pernet operations structure for the subsystem
 *
 *      Register a subsystem which has init and exit functions
 *      that are called when network namespaces are created and
 *      destroyed respectively.
 *
 *      When registered all network namespace init functions are
 *      called for every existing network namespace.  Allowing kernel
 *      modules to have a race free view of the set of network namespaces.
 *
 *      When a new network namespace is created all of the init
 *      methods are called in the order in which they were registered.
 *
 *      When a network namespace is destroyed all of the exit methods
 *      are called in the reverse of the order with which they were
 *      registered.
```


接下來看一下這個函式的呼叫範例
```c=
	ret = register_pernet_subsys(&ipvs_core_ops);	/* Alloc ip_vs struct */
```

其傳入的參數是一個名為 `ipvs_core_ops` 的結構，而該結構內容如下

```c＝
static struct pernet_operations ipvs_core_ops = {
	.init = __ip_vs_init,
	.exit = __ip_vs_cleanup,
	.id   = &ip_vs_net_id,
	.size = sizeof(struct netns_ipvs),
};
```

這個結構內最重要的就是 **.init** 這個 **function pointer**，其意思是 **每當有任何一個 namespace 被創建之時**，請呼叫對應的 **function** 來處理，而這邊這個 **function** 就是 **__ip_vs_init**.

```c=
/*
 *	Initialize IP Virtual Server netns mem.
 */
static int __net_init __ip_vs_init(struct net *net)
{
	struct netns_ipvs *ipvs;
	int ret;

	ipvs = net_generic(net, ip_vs_net_id);
	if (ipvs == NULL)
		return -ENOMEM;

	/* Hold the beast until a service is registerd */
	ipvs->enable = 0;
	ipvs->net = net;
	/* Counters used for creating unique names */
	ipvs->gen = atomic_read(&ipvs_netns_cnt);
	atomic_inc(&ipvs_netns_cnt);
	net->ipvs = ipvs;

	if (ip_vs_estimator_net_init(ipvs) < 0)
		goto estimator_fail;

	if (ip_vs_control_net_init(ipvs) < 0)
		goto control_fail;

	if (ip_vs_protocol_net_init(ipvs) < 0)
		goto protocol_fail;

	if (ip_vs_app_net_init(ipvs) < 0)
		goto app_fail;

	if (ip_vs_conn_net_init(ipvs) < 0)
		goto conn_fail;

	if (ip_vs_sync_net_init(ipvs) < 0)
		goto sync_fail;

	ret = nf_register_net_hooks(net, ip_vs_ops, ARRAY_SIZE(ip_vs_ops));
	if (ret < 0)
		goto hook_fail;

	return 0;
/*
 * Error handling
 */

hook_fail:
	ip_vs_sync_net_cleanup(ipvs);
sync_fail:
	ip_vs_conn_net_cleanup(ipvs);
conn_fail:
	ip_vs_app_net_cleanup(ipvs);
app_fail:
	ip_vs_protocol_net_cleanup(ipvs);
protocol_fail:
	ip_vs_control_net_cleanup(ipvs);
control_fail:
	ip_vs_estimator_net_cleanup(ipvs);
estimator_fail:
	net->ipvs = NULL;
	return -ENOMEM;
}
```

該函式的註解直接標示，針對每一個 `network namespace` 去進行相關的初始化，其中為重要的則是 `ip_vs_control_net_init` 以及 `nf_register_net_hooks` 這兩個函式，後者則是與 **netfilter** 也就是 **iptables** 相關的互動，下一章節我們再來仔細看一下這個函式。

## ip_vs_control_net_init
根據上述的 **debug** 規則我們知道系統上會有一個路徑 **/proc/sys/net/ipv4/vs** 可以讓使用者與之互動，而這個介面的初始化其實就是透過 **ip_vs_control_net_init** 來完成的。

```c++
int __net_init ip_vs_control_net_init(struct netns_ipvs *ipvs)
{
        int i, idx;
...
...
        if (ip_vs_control_net_init_sysctl(ipvs))
                goto err;
...
```

其呼叫了 `ip_vs_control_net_init_sysctl`
```c++
#ifdef CONFIG_SYSCTL
static int __net_init ip_vs_control_net_init_sysctl(struct netns_ipvs *ipvs)
{
        struct net *net = ipvs->net;
        int idx;
        struct ctl_table *tbl;
...
        idx = 0;
...
        tbl[idx++].data = &ipvs->sysctl_backup_only;
        ipvs->sysctl_conn_reuse_mode = 1;
        tbl[idx++].data = &ipvs->sysctl_conn_reuse_mode;
        tbl[idx++].data = &ipvs->sysctl_schedule_icmp;
        tbl[idx++].data = &ipvs->sysctl_ignore_tunneled;
...
        ipvs->sysctl_hdr = register_net_sysctl(net, "net/ipv4/vs", tbl);
...
    return 0
}
```
上述的概念就是創建一個 **struct ctl_table *tbl** 的物件，並且填入相關的資訊後，呼叫 **register_net_stsctl** 來註冊這一系列的路徑，也可以觀察到其子路徑就是 **net/ipv4/vs**. 而這些 **table** 的資料的定義如下

```c++
/*
 *      IPVS sysctl table (under the /proc/sys/net/ipv4/vs/)
 *      Do not change order or insert new entries without
 *      align with netns init in ip_vs_control_net_init()
 */

static struct ctl_table vs_vars[] = {
        {
                .procname       = "amemthresh",
                .maxlen         = sizeof(int),
                .mode           = 0644,
                .proc_handler   = proc_dointvec,
        },
        {
                .procname       = "am_droprate",
                .maxlen         = sizeof(int),
                .mode           = 0644,
                .proc_handler   = proc_dointvec,
        },
        {
                .procname       = "drop_entry",
                .maxlen         = sizeof(int),
                .mode           = 0644,
                .proc_handler   = proc_do_defense_mode,
        },
....
#ifdef CONFIG_IP_VS_DEBUG
        {
                .procname       = "debug_level",
                .data           = &sysctl_ip_vs_debug_level,
                .maxlen         = sizeof(int),
                .mode           = 0644,
                .proc_handler   = proc_dointvec,
        },
#endif
        { }
};
```

這邊用一種格式化的方式去定義每個路徑的 **名稱, 類別， 相關的處理函式以及存取模式**，特別注意到的是 **debug_level** 本身則是被 **ifdef CONFIG_IP_VS_DEBUG** 這個 **macro** 給包起來，所以如果沒有特別處理的話，預設情況下就不會把 **debug_level** 給編譯進去。

觀看這些資料還有一個好處就是你可以知道系統中有哪些參數可以餵給 Kernel 去處理，同時也可以搭配觀看原始碼的方式來了解這些參數到底實際上做了什麼。


這邊有一個概念要注意的就是 **net** 這個物件是 **kernel** 內網路系統最重要的結構，每個 **netowrk namespace** 都會有一個自己的副本，基本上只要函數本身有吃 **net** 參數，就可以猜到這個功能絕對是每個 **network namespace** 獨立。這邊要特別注意的是 **Host** 本身也是一個 **network namespace**，所以當我們註冊這個函式的時候，就會先針對系統本身這個 **network namespace** 呼叫這一組對應的 **init function**.


如果你的操作環境是舊版一點的 `kernel` ，譬如 `4.4.0` 之類的，你可以在系統觀察到類似下列的訊息
```
[3534469.163231] IPVS: Creating netns size=2192 id=1342
[3534793.388007] IPVS: Creating netns size=2192 id=1343
[3535052.371673] IPVS: Creating netns size=2192 id=1344
[3535706.550968] IPVS: Creating netns size=2192 id=1345
[3535761.378872] IPVS: Creating netns size=2192 id=1346
[3537083.860486] IPVS: Creating netns size=2192 id=1347
```

題外話， `kernel code` 內滿滿的 `goto`, `goto` 不是不能用，而是你要會用，可以看看 **kernel** 這邊的用法，同時你也可以想想不用 **goto** 的話，這些程式碼你會怎麼修改。

# Summary

今天這個章節針對 **IPVS** 底層的資訊進行了粗略地探索，主要是知道要如何透過 **IPVS** 提供的方式來進行除錯，藉由這除錯方式之後會更方便地去整個瞭解程式的呼叫脈絡並且我認為也是一個很好的學習方式

對於 **IPVS** 的概念可以理解成，**IPVS** 的功能由 **Kernel** 提供，但是每個 **network namespace** 互相獨立不影響彼此，所以你到不同的 **network namespace** 內透過 **ipvsadm** 看到的結果都互相獨立，不會牽扯彼此。

透過註冊 **register_pernet_subsys** 這個函式， **ipvs** 能夠自動對系統上每個已經存在/未來新增的 **network namespace** 進行處理，幫其初始化 **ipvs** 相關的物件。

最後，由於 **init** 函式開始後就會去進行 **sys** 相關的初始化，來不及將 **debug_level** 設定而輸出，因此我在修改原始碼之後重新安裝到系統來觀察更細部的 **init** 流程。

```bash=
[669447.783857] IPVS: Enter: ip_vs_init, net/netfilter/ipvs/ip_vs_core.c line 2371
[669447.783858] IPVS: Enter: ip_vs_control_init, net/netfilter/ipvs/ip_vs_ctl.c line 4122
[669447.783862] IPVS: Enter: ip_vs_protocol_init, net/netfilter/ipvs/ip_vs_proto.c line 338
[669447.783863] IPVS: Registered protocols (TCP, UDP, SCTP, AH, ESP)
[669447.783864] IPVS: Enter: ip_vs_conn_init, net/netfilter/ipvs/ip_vs_conn.c line 1403
[669447.783937] IPVS: Connection hash table configured (size=4096, memory=64Kbytes)
[669447.783937] IPVS: Each connection entry needs 288 bytes at least
[669447.783944] IPVS: Enter: __ip_vs_init, net/netfilter/ipvs/ip_vs_core.c line 2257
[669447.783945] IPVS: Enter: ip_vs_control_net_init, net/netfilter/ipvs/ip_vs_ctl.c line 4037
[669447.783951] IPVS: Enter: ip_vs_control_net_init_sysctl, net/netfilter/ipvs/ip_vs_ctl.c line 3926
[669447.783963] IPVS: Enter: ip_vs_protocol_net_init, net/netfilter/ipvs/ip_vs_proto.c line 309
[669447.783966] IPVS: Enter: ip_vs_conn_net_init, net/netfilter/ipvs/ip_vs_conn.c line 1384
[669447.783968] IPVS: Enter: __ip_vs_dev_init, net/netfilter/ipvs/ip_vs_core.c line 2329
[669447.981664] IPVS: Enter: ip_vs_register_nl_ioctl, net/netfilter/ipvs/ip_vs_ctl.c line 4091
[669447.981678] IPVS: ipvs loaded.
```

下一章節我們會主要針對 封包的流向 來進行探討，包含 **iptables/netfilter** 與之的互動，以及實際由本地端送出一個封包後，實際上背後的運作邏輯是哪些，又會呼叫到哪些 **function** 來處理。


# 課程分享

最後，我目前於 Hiskio 上面有開設一門 Kubernetes 入門篇的課程，裡面會探討運算/網路/儲存三個最重要的平台資源，此外對於 CRI/CNI/CSI 也都有簡單的介紹，主要會基於 **Kubernetes** 本身的設計原理及各資源的用法與情境去介紹。
如果本身已經很熟練的使用 Kubernetes 於環境中就不太適合這門課程，主要是給想要踏入到 Kubernetes 世界中的朋友，有興趣的幫忙捧場或推廣
組合包
https://hiskio.com/packages/D7RZGWrNK

單堂(CI/CD)
https://hiskio.com/courses/385?promo_code=13K49YE&p=blog1

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE

# Reference
http://www.programmersought.com/article/907433913/
