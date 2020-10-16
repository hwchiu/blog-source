---
title: IPvS 學習手冊(四)
tags:
  - IPVS
  - Network
  - Linux
  - Kernel
keywords: 'Kubernetes,Network,Linux,Ubuntu'
description: >-
  本文作為 IPVS 系列文第四篇，主要是跟大家介紹 IPVS 於 Linux Kernel 內的架構設計，透過理解其設計更可以幫助我們去瞭解 IPVS 與
  IPTABLES 的差異，面對諸如此類的探討文章更能夠有足夠的背景去思考與學習
abbrlink: 59071
date: 2020-08-16 06:48:44
---



# Preface

本篇文章作為系列文章的第四篇，該系列文希望能夠從概念到實作，從簡單到複雜來探討 IPVS (IP Virtual Server) 的概念，目前規劃的主題包含：

- [IPVS 的基本使用與概念](https://www.hwchiu.com/ipvs-1.html)
- [IPVS 與 Kubernetes 的整合](https://www.hwchiu.com/ipvs-2.html)
- [IPVS 除錯方式與基本 Kernel Module 概念](https://www.hwchiu.com/ipvs-3.html)
- [IPVS Kernel 架構實現](https://www.hwchiu.com/ipvs-4.html)

本文主要是從 Linux Kernel 出發，探討一下 IPVS 於 Kernel 的實作，並且將其與 IPTABLES 的實作進行比較，透過這些實作的差異，更可以理解在效能上為何兩者會有所差異

# 環境
整篇文章都是基於 Linux Kernel 4.15 為基準去閱讀, 可以從 **[Github](https://github.com/torvalds/linux/tree/v4.15/net/netfilter/ipvs)** 或是 **[LXR](https://elixir.bootlin.com/linux/v4.15/source)** 來進行線上閱讀

# Architecture
過往在使用 IPVS 的時候，都會透過工具 `ipvsadm` 來進行管理，而這個管理工具實際上本身只有設定的功用，真正的封包處理都是在 Kernel 內處理，更直接的可以說整個 Linux Kernel 就是一個運行的大伺服器，而 IPVS 則是可透過 **Kernel Module** 動態開啟的功能。

接下來將會使用下列示意圖作為基底，來參考學習到底 IPVS 的底層實作概念

![](https://i.imgur.com/kHm5pXm.png)

## User Space

就如同上述所說， `user space` 的工具就是 `ipvsadm`, 其本身都是透過 **get/set socketopt** 這個方式與底層溝通，其相關的原始碼可以參考這份[專案](https://github.com/formorer/pkg-ipvsadm)

其中一個有趣的地方在於， **ipvsadm** 本身使用上不需要先自己安裝相關的 **Kerenl Module**，這部分是因為 **ipvsadm** 的原始碼裡面會先幫忙檢查當前系統是否已經存在對應的 **ipvs.ko**，如果不存在，則透過 `modprobe` 這個工具來安裝 `ip_vs`。
也是因為透過這個機制，大部分的 Linux 機器如果要成為 Kubernetes 節點並且啟動 **IPVS** 的功能，都不需要事先去安裝所需的 **kernel module**，對於系統設定來說省了一些工

以下內容都可以於 [ipvsadm.c](https://github.com/formorer/pkg-ipvsadm/blob/master/ipvsadm.c) 中找到
```c=
int main(int argc, char **argv)
{
	int result;

	if (ipvs_init()) {
		/* try to insmod the ip_vs module if ipvs_init failed */
		if (modprobe_ipvs() || ipvs_init())
			fail(2, "Can't initialize ipvs: %s\n"
				"Are you sure that IP Virtual Server is "
				"built in the kernel or as module?",
			     ipvs_strerror(errno));
	}

	/* warn the user if the IPVS version is out of date */
	check_ipvs_version();

	/* list the table if there is no other arguement */
	if (argc == 1){
		list_all(FMT_NONE);
		ipvs_close();
		return 0;
	}

	/* process command line arguments */
	result = process_options(argc, argv, 0);

	ipvs_close();
	return result;
}
```

```c=
static int modprobe_ipvs(void)
{
	char *argv[] = { "/sbin/modprobe", "--", "ip_vs", NULL };
	int child;
	int status;
	int rc;

	if (!(child = fork())) {
		execv(argv[0], argv);
		exit(1);
	}

	rc = waitpid(child, &status, 0);

	if (rc == -1 || !WIFEXITED(status) || WEXITSTATUS(status)) {
		return 1;
	}

	return 0;
}
```

## Kernel Space

**IPVS** 最主要的功能都是由 **ip_vs.ko** 這個模組提供的，該模組的功能粗略可以分成三大項
1. 處理與 **UserSpace** 的溝通，譬如提供接口供 **Set/Get Socketopt** 介面使用
2. 處理與 **Schduleing Algorithm** 的溝通，每個 **Schduleing** 本身都是一個獨立的 **kernel module**，譬如 **ipvs_rr.ko**
    - **ipvs.ko** 本身也會自動插入對應的 kernel module，所以使用者並不需要事先安裝，只要確保系統上有相關的 **kernel module** 檔案，而這部分 **Ubuntu** 發行版本都有，其餘的發行版本則不確定
3. 真正處理封包的核心邏輯，這部分分成 **什麼時間處理封包** 以及 **怎麼處理封包**
    - **ipvs.ko** 本身包含了眾多的 **function** 來解決 **怎麼處理封包**，這部分包含查找對應的資料結構，呼叫對應的 **scheduling algorithm** 選擇後端伺服器以及封包轉發
    - **netfilter** 的架構則提供了 **什麼時間處理封包**，這部分與常見的 **IPTABLES** 時間點一樣，譬如 **PREROUTING**, **POSTROUTING**, **INPUT**, **OUTPUT** 等

實際上，(3)處理封包核心邏輯的實作則是將兩個概念結合， **ipvs.ko** 將處理封包的 function 透過 Linux Kernel 的介面去跟 netfilter 註冊，要求當封包經過特定的 **HOOK** 點的時候，呼叫相關的 function 來處理封包

下圖是一個基本的 **IPTABLES/Netfilter** 流程圖，基本上就是**五個**時間點，每個時間點內都會有不同對應的 **TABLE**，而這些 **TABLE** 內也有相對應的 **Rules**
實際上的運作流程就是封包到達這些時間點後，會依序跳到個 **TABLE** 裡面，並且依序執行 **RULES**。
![](https://i.imgur.com/GymG1h1.png)


下圖則是將 **IPVS** 與 **IPTABLES/Netfilter** 結合後的圖片，差異性就是**五個**時間點內中有三個時間點變動，分別是 **INPUT/OUTPUT/FORWARD**.
**ipvs.ko** 針對這三個時間點分別去註冊三個不同的 **Function**，而這些 **Function** 的執行點都不同。
![](https://i.imgur.com/M9aIyuT.png)

1. INPUT:
註冊的 function 會在 **Filter TABLE** 後執行，而 function 完畢後則會跳去執行 **NAT TABLE**.
2. OUTPUT:
註冊的 function 會在 **nat TABLE** 後執行，而 function 完畢後則會跳去執行 **filter TABLE**.
3. FORWARD:
註冊的 function 會在 **Filter TABLE** 後執行，而 function 完畢後則會結束整個 **Forward** 時間點的處理，讓 Netfilter 繼續往下處理

這邊針對整個架構做一個總結
1. UserSpace 的工具會透過事先定義好的介面與 Kernel 溝通，而定義該介面的則是 **ip_vs.ko** 這個 Kernel Module
2. **ip_vs.ko** 做非常多事情，包含了
    a. 定義封包處理的 function，並且透過 netfilter 框架註冊這些 function. 未來封包在 kernel 內遊走的時候就會被這些 function 處理
    b. 根據參數設立相關的資料結構，包含有哪些 backend server, 採用哪套 scheduling 演算法
    c. 根據需求動態載入對應 scheduling 的 kernel module.


接下來我們就會從原始碼的部分稍微看一下 **ip_vs.ko** 這些事情實際上是怎麼實作的


# Source Code
原始碼的部分，基於 Linux Kernel 4.15 為基準去閱讀, 可以從 **[Github](https://github.com/torvalds/linux/tree/v4.15/net/netfilter/ipvs)** 或是 **[LXR](https://elixir.bootlin.com/linux/v4.15/source)** 來進行線上閱讀

## 載入 Scheduling Kernel Module
這部分的程式碼都在 [ip_vs_sched.c](https://elixir.bootlin.com/linux/v4.15/source/net/netfilter/ipvs/ip_vs_sched.c) 裡面，重要的有幾個
1. 提供介面根據字串去找出對應的 Scheduler, 如果不存在，就嘗試幫忙載入
2. 提供介面給呼叫者去註冊，闡明自己是個 Scheduler

1)的部分主要是下列的程式碼，首先嘗試取得，如果取得失敗就去幫忙載入，並且再次讀取
```c=
/*
 *  Lookup scheduler and try to load it if it doesn't exist
 */
struct ip_vs_scheduler *ip_vs_scheduler_get(const char *sched_name)
{
	struct ip_vs_scheduler *sched;

	/*
	 *  Search for the scheduler by sched_name
	 */
	sched = ip_vs_sched_getbyname(sched_name);

	/*
	 *  If scheduler not found, load the module and search again
	 */
	if (sched == NULL) {
		request_module("ip_vs_%s", sched_name);
		sched = ip_vs_sched_getbyname(sched_name);
	}

	return sched;
}
```
2)的部分如下，基本上就是一些資料結構的處理，呼叫者則會是那些不同算法的 Scheduler
```c=
int register_ip_vs_scheduler(struct ip_vs_scheduler *scheduler)
{
	struct ip_vs_scheduler *sched;

	if (!scheduler) {
		pr_err("%s(): NULL arg\n", __func__);
		return -EINVAL;
	}

	if (!scheduler->name) {
		pr_err("%s(): NULL scheduler_name\n", __func__);
		return -EINVAL;
	}
.....
}
```

基本上所有的 IPVS Scheduling 的實作都會再 Kernel Module 初始化的過程去呼叫 **register_ip_vs_scheduler**，舉例來說 [ip_vs_rr.c](https://elixir.bootlin.com/linux/v4.15/source/net/netfilter/ipvs/ip_vs_rr.c) 這個 kernel module 被系統載入後，第一件事情就是呼叫 **register_ip_vs_scheduler**，將自己的物件註冊進去，所以之後有任何人透過 **ipvsadm** 去表示我要使用 **rr** 這個算法的時候，底下的 kernel module(ip_vs.ko) 就有辦法找到對應的 scheduler 來使用。

```c=

static int __init ip_vs_rr_init(void)
{
	return register_ip_vs_scheduler(&ip_vs_rr_scheduler);
}

static void __exit ip_vs_rr_cleanup(void)
{
	unregister_ip_vs_scheduler(&ip_vs_rr_scheduler);
	synchronize_rcu();
}

module_init(ip_vs_rr_init);
module_exit(ip_vs_rr_cleanup);
MODULE_LICENSE("GPL");
```

## 註冊 Netfilter Function

**ip_vs** 這個 Kernel Module 本身被載入後，也會有相關的初始步驟，該步驟會其實是呼叫的 `__ip_vs_init` 這個函式
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

這邊最重要的就是 `nf_register_net_hooks` 這個函式，該函式會傳入 **ip_vs_ops** 這個物件，而這個函式就是跟 `netfilter` 去註冊，接下來實際看一下 **ip_vs_ops** 物件(忽略 ipv6, 因為跟 ipv4 概念雷同)



```c=
static const struct nf_hook_ops ip_vs_ops[] = {
	/* After packet filtering, change source only for VS/NAT */
	{
		.hook		= ip_vs_reply4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC - 2,
	},
	/* After packet filtering, forward packet through VS/DR, VS/TUN,
	 * or VS/NAT(change destination), so that filtering rules can be
	 * applied to IPVS. */
	{
		.hook		= ip_vs_remote_request4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC - 1,
	},
	/* Before ip_vs_in, change source only for VS/NAT */
	{
		.hook		= ip_vs_local_reply4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_NAT_DST + 1,
	},
	/* After mangle, schedule and forward local requests */
	{
		.hook		= ip_vs_local_request4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_NAT_DST + 2,
	},
	/* After packet filtering (but before ip_vs_out_icmp), catch icmp
	 * destined for 0.0.0.0/0, which is for incoming IPVS connections */
	{
		.hook		= ip_vs_forward_icmp,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_FORWARD,
		.priority	= 99,
	},
	/* After packet filtering, change source only for VS/NAT */
	{
		.hook		= ip_vs_reply4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_FORWARD,
		.priority	= 100,
	},
...
```
ip_vs_ops 是一個 **nf_hook_ops** 的陣列物件，每個物件內都有四個成員被使用，分別是
1. hook
2. pf
3. hooknum
4. priority

這四個組合起來的含義就是，我準備了一個 **hook** 的函式，希望當封包符合 **pf** 描述的格式，且封包目前運行到 **hooknum** 這個時間點的時候，根據我描述的 **proiroty** 於正確的時間點呼叫我準備的 **hook** 函式。

我們取其中一個範例來進行比較詳細的解釋
```c=
	{
		.hook		= ip_vs_reply4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC - 2,
	},
```

首先，`hook` 的值是 `ip_vs_reply4`，這個其實是一個 **function**，內容如下
```c=
/*
 *	It is hooked at the NF_INET_FORWARD and NF_INET_LOCAL_IN chain,
 *	used only for VS/NAT.
 *	Check if packet is reply for established ip_vs_conn.
 */
static unsigned int
ip_vs_reply4(void *priv, struct sk_buff *skb,
	     const struct nf_hook_state *state)
{
	return ip_vs_out(net_ipvs(state->net), state->hook, skb, AF_INET);
}
```

`pf` 的值則是 **NFPROTO_IPV4**，其他相關的變數則定義於 [netfilter.h](https://elixir.bootlin.com/linux/v4.15/source/include/uapi/linux/netfilter.h#L67)
```c=
enum {
	NFPROTO_UNSPEC =  0,
	NFPROTO_INET   =  1,
	NFPROTO_IPV4   =  2,
	NFPROTO_ARP    =  3,
	NFPROTO_NETDEV =  5,
	NFPROTO_BRIDGE =  7,
	NFPROTO_IPV6   = 10,
	NFPROTO_DECNET = 12,
	NFPROTO_NUMPROTO,
};
```

`hooknum` 則是我們一直強調的時間點，對應到 **netfilter/iptables** 的架構圖，就是 **LOCAL_IN**，封包經由 **routing** 後決定要送到本機處理的時間點，其他相關的變數則定義於
 [netfilter.h](https://elixir.bootlin.com/linux/v4.15/source/include/uapi/linux/netfilter.h#L67)
```c=
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS
};
```
可以由上面的原始碼看到這些變數的名稱都與我們熟悉的 **IPTABLES** 用法熟悉，這是因為都基於 **Netfilter** 架構的原因

最後的 **priority** 則是一個優先度，數值愈小代表優先度愈高，也先執行，其他相關的變數則定義於[netfilter_ipv4.h](https://elixir.bootlin.com/linux/v4.15/source/include/uapi/linux/netfilter_ipv4.h#L68)
```c=
enum nf_ip_hook_priorities {
	NF_IP_PRI_FIRST = INT_MIN,
	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
	NF_IP_PRI_RAW = -300,
	NF_IP_PRI_SELINUX_FIRST = -225,
	NF_IP_PRI_CONNTRACK = -200,
	NF_IP_PRI_MANGLE = -150,
	NF_IP_PRI_NAT_DST = -100,
	NF_IP_PRI_FILTER = 0,
	NF_IP_PRI_SECURITY = 50,
	NF_IP_PRI_NAT_SRC = 100,
	NF_IP_PRI_SELINUX_LAST = 225,
	NF_IP_PRI_CONNTRACK_HELPER = 300,
	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
	NF_IP_PRI_LAST = INT_MAX,
};
```

透過這些資訊，我們可以瞭解到上述的含義就是請幫我於 **LOCAL_IN** 這個時間點註冊一個 function(ip_vs_reply4)，若封包是 IPV4 的格式且請比 **NAT_SRC(SOURCE NAT)** 還要提早處理。
```c=
	{
		.hook		= ip_vs_reply4,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC - 2,
	},
```

這也是為什麼我們架構圖會於 **INPUT(LOCAL_IN)** 裡面將 **IPVS** 放到 **NAT** 前面執行
![](https://i.imgur.com/M9aIyuT.png)

其他的 **FORWARD**, **OUTPUT(LOCAL_OUT)** 的概念是完全一樣的

這邊還有一個有趣的概念就是，其實這些 **Talbe (NAT/FILTER/MANGLE)** 的本質都是一個函式，舉例來說 [iptable_net.c](https://elixir.bootlin.com/linux/v4.15/source/net/ipv4/netfilter/iptable_nat.c#L34)  裡面就定義了 **NAT** 相關的操作
```c=
static const struct nf_hook_ops nf_nat_ipv4_ops[] = {
	/* Before packet filtering, change destination */
	{
		.hook		= iptable_nat_ipv4_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= iptable_nat_ipv4_out,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
	/* Before packet filtering, change destination */
	{
		.hook		= iptable_nat_ipv4_local_fn,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= iptable_nat_ipv4_fn,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
};
```
，而所謂的 **NAT Table** 的概念其實就被上述的 **hook** 給處理掉，如下圖，其中 **ipt_do_table** 作為一個 **function pointer** 被上述的 **hook** 裡面使用，而這個函式內則是會透過 `do...while` 去依序執行相關的規則
```c=
static unsigned int iptable_nat_do_chain(void *priv,
					 struct sk_buff *skb,
					 const struct nf_hook_state *state,
					 struct nf_conn *ct)
{
	return ipt_do_table(skb, state, state->net->ipv4.nat_table);
}

static unsigned int iptable_nat_ipv4_fn(void *priv,
					struct sk_buff *skb,
					const struct nf_hook_state *state)
{
	return nf_nat_ipv4_fn(priv, skb, state, iptable_nat_do_chain);
}

static unsigned int iptable_nat_ipv4_in(void *priv,
					struct sk_buff *skb,
					const struct nf_hook_state *state)
{
	return nf_nat_ipv4_in(priv, skb, state, iptable_nat_do_chain);
}

static unsigned int iptable_nat_ipv4_out(void *priv,
					 struct sk_buff *skb,
					 const struct nf_hook_state *state)
{
	return nf_nat_ipv4_out(priv, skb, state, iptable_nat_do_chain);
}

static unsigned int iptable_nat_ipv4_local_fn(void *priv,
					      struct sk_buff *skb,
					      const struct nf_hook_state *state)
{
	return nf_nat_ipv4_local_fn(priv, skb, state, iptable_nat_do_chain);
}
```

其他的 **table** 概念一樣，只是使用的介面不太一樣，都改使用 **xt** 系列的 API來處理


# IPTABLES 比較
藉由上述原始碼的觀察，我們可以觀察到一個很重要的事情就是 **IPVS** 本身的運作是依賴 **netfilter** 架構來處理封包，而很剛好的是 **IPTABLES** 本身也是大量依賴 **netfilter** 來使用，這邊的比較我認為有幾個重點
## IPVS 與 IPTABLES 的取代問題
我認為沒有辦法將 IPVS 與 IPTABLES 直接評比優劣，做出誰能夠取代誰的結論。
原因是兩者個功能面向完全不同， IPVS 能夠提供的功能很少，就是完全針對 LoadBalancing 實作而已，然而 IPTABLES 本身可以做的事情非常多，特別是透過 `-m`, `-j` 這些 module擴充後可以辦到的事情更多了。

大部分的評比文章其實都是針對於 **Load-Balancing** 這件事情來比較，如果真的夠熟悉其底層架構就會理解到， **IPVS** 是專門針對 **LB** 去實作的，然而 **IPTABLES** 並不是，而是採用一些比較獵奇的方式來達到類似的功能
因此效能上會有差異也不足為奇，甚至說若 IPVS 沒有更好，根本沒有發展的必要(我認為)。

## IPVS 除錯不易
綜觀 IPVS 的使用文件與架構說明，基本上都不太會講到 IPVS 與 Netfilter 的關係，但是其實對於每個封包來說，其實都是基於 **netfilter** 的架構在跑，會先被 **iptables** 的規則處理(mangle/filter)，接下來才被 IPVS 擷取去處理。
這種情況下，當發現問題時，其實不太容易知道問題點在哪裡，到底是 **IPTABLES** 的規則出錯，還是 **IPVS** 本身的功能或是設定出錯。
此外，對於 IPVS 使用者來說能夠觀察到的資訊大部分都是依賴 **ipvsadm** 這個工具，顯示的就只有 **設定了哪些 server**, **分配了多少封包** 等比較上層的資訊，對於除錯能夠提供的線索有限，最終還是要連同 **iptables** 一起觀看研究，才能夠鎖定問題的發生點。


# 結論

拖了很久的第四篇 **IPVS** 文章終於寫完，這中間也經歷了一次的線上 Meetup 來濃縮四篇文章的介紹，該次 Meetup 也有[線上錄影](https://www.youtube.com/watch?v=6ZaUIRZ-SO8&t=14s), 有興趣的也可以觀看

{% youtube  6ZaUIRZ-SO8 %}

藉由這四篇文章的學習，讓我對於 IPVS 的概念有更深的理解，同時也更能說出 IPVS 與 IPTABLES 的異同之處，對於相關文章也能夠有更深的背景去探討與思考。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/D7RZGWrNK

單堂(CI/CD)
https://hiskio.com/courses/385?promo_code=13K49YE&p=blog1

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>