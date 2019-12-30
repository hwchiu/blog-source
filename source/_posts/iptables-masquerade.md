---
title: 'Linux NAT Masquerade 研究(上)'
keywords: 'Network,Linux,Ubuntu,iptables,nat,snat,masquerade'
abbrlink: 56309
date: 2019-12-30 05:43:35
tags:
  - Network
  - Linux
  - Iptables
description: 本篇文章透過閱讀原始碼的方式來學習 MASQUERADE 的運作模式，而 MASQUERADE 則是被廣為使用的 SOURCE NAT 模組。作為一個 IPTABLES 的擴充模組，透過觀察原始碼的方式可以學習到是如何處理相關的參數甚至，選擇來源 IP 地址以及來源連接埠等相關行為
---

# Preface
本篇文章主要用來探討 **Linux** 內最常使用的 **SNAT** 模組 **MASQUERADE** 與多個 **IP** 地址交互使用時遇到的問題。

對於 **MASQUERADE** 這個指令來說，一般最常見的使用情境就是動態 **Source NAT**，根據情況動態修改封包的來源地址，譬如下列指令
```
-A POSTROUTING -s 172.18.0.0/16 ! -o docker0 -j MASQUERADE
```
與之相對的另外一個指令則是靜態的 **Source NAT** 功能，
```
-A POSTROUTING -o eth0 -j SNAT --to 1.2.3.4
```
明確的指出封包轉換後的來源地址，就我個人的經驗來說，這個功能比較少被使用，主要是要寫死出去的封包地址，因此一旦系統上關於 **IP** 地址有任何修改的時候，相關的規則全部都要重新調整。

實務上這也牽扯到滿多系統的設定，因此大部分的情境都會喜歡使用 **MASQUERADE** 的方式，讓 **Linux Kernel** 來根據系統當前設定來選擇要使用的 **IP** 地址。

而這篇文章的誕生起因非常簡單，我於某天遇到了一個情況，一個網卡介面本身被指定了多個**IP** 地址，那這種情況下 **MASQUERADE** 會怎麼運作，會怎麼挑選?

接下來的文章就會針對 **MASQUERADE** 的實現進行探討並且嘗試回答上述的問題

# 環境設定
- [Linux Kernel 4.15.18](https://elixir.bootlin.com/linux/v4.15.18/source
)
- [iptables  v.1.6.1](https://git.netfilter.org/iptables/tree/extensions/libip6t_MASQUERADE.c?h=v1.6.1)
    
# 問題描述

**MASQUERADE** 最初的認知，會針對不同的 **Routing** 規則選出不同的網路介面作為目的地，並且選擇該網路介面本身的 **IP** 地址作為封包最後的來源地址。

但是如果今天一個網路介面本身有多個 **IP** 地址，那 **MASQUERADE** 這個功能到底會怎麼去選擇 **IP** 地址？ 此外到底是怎麼選擇相關的連接埠 **source port** ?

針對這個問題，我決定採用兩個步驟來研究這個問題，首先我們先針對 **MASQUERADE** 的架構以及原始碼來進行研究進行分析。

接者透過修改 **MASQUERADE** 原始碼的方式來加入一些除錯訊息幫助我們釐清其運作原理，並且佐證上述的猜測(下篇)。

# 架構分析
**MASQUERADE** 本身不是一個獨立運作的元件與程式，而是一個基於 **netfilter/iptables** 框架而開發的模組，因此在使用上必須搭配 **iptables** 的指令來使用，這也是普遍大家使用上最初接觸的方式
```bash
iptables -t nat -j MASQUERADE
```

由於整個架構都是跟者 **netfilter/iptables** 一起，所以 **MASQUERADE** 本身的架構也必須遵照上述的規範。整個程式碼分成兩大部分，分別是 **User Space** 的參數控制端以及 **Kernel Space**的邏輯執行端

## UserSpace
原始碼的部分請參閱此檔案 [iptables  v.1.6.1](https://git.netfilter.org/iptables/tree/extensions/libip6t_MASQUERADE.c?h=v1.6.1)

**User Space** 的部分是一個依賴於 **iptables** 框架的模組，主要功能就是擴充 **iptables** 這個指令，能夠支援不同的參數與用法並且將該規則輸入的參數順利地記住下來，之後會傳入到 **kernel space** 中交給真正負責處理封包的邏輯區塊去處理。

接下來簡單的看一下這個檔案的內容，快速的看一下即可
```c++
static struct xtables_target masquerade_tg_reg = {
        .name           = "MASQUERADE",
        .version        = XTABLES_VERSION,
        .family         = NFPROTO_IPV4,
        .size           = XT_ALIGN(sizeof(struct nf_nat_ipv4_multi_range_compat)),
        .userspacesize  = XT_ALIGN(sizeof(struct nf_nat_ipv4_multi_range_compat)),
        .help           = MASQUERADE_help,
        .init           = MASQUERADE_init,
        .x6_parse       = MASQUERADE_parse,
        .print          = MASQUERADE_print,
        .save           = MASQUERADE_save,
        .x6_options     = MASQUERADE_opts,
        .xlate          = MASQUERADE_xlate,
};

void _init(void)
{
	xtables_register_target(&masquerade_tg_reg);
}
```

對於每個 **iptable module(xtables)** 來說都必須要實現一個相關的結構，並且針對每個 **function pointer** 設定相關的處理函式。
並且透過一個框架所提供的函式去註冊這個結構，只要這邊成功處理，就可以透過 **iptables** 的指令來操作這個擴充功能 **MASQUERADE**。

這裡面有滿多的函式，就針對比較重要的來看，也就是針對參數處理的部分 **x6_parse(**MASQUERADE_parse**)** 來看。

```c++
static const struct xt_option_entry MASQUERADE_opts[] = {
	{.name = "to-ports", .id = O_TO_PORTS, .type = XTTYPE_STRING},
	{.name = "random", .id = O_RANDOM, .type = XTTYPE_NONE},
	XTOPT_TABLEEND,
};
```
對於每個 **iptables** 的擴充模組來說，都可以透過定義 **xt_option_entry** 類別來定義要使用的參數以及型態，所以其實 **MASQUERADE** 本身還有兩個參數可以使用，分別是 **to-ports** 以及 **random**，至於真正的含義以及用途就要看 **help** 或是直接看底層的實現來了解。

```c++
static void MASQUERADE_parse(struct xt_option_call *cb)
{
        const struct ipt_entry *entry = cb->xt_entry;
        int portok;
        struct nf_nat_ipv4_multi_range_compat *mr = cb->data;

        if (entry->ip.proto == IPPROTO_TCP
            || entry->ip.proto == IPPROTO_UDP
            || entry->ip.proto == IPPROTO_SCTP
            || entry->ip.proto == IPPROTO_DCCP
            || entry->ip.proto == IPPROTO_ICMP)
                portok = 1;
        else
                portok = 0;

        xtables_option_parse(cb);
        switch (cb->entry->id) {
        case O_TO_PORTS:
                if (!portok)
                        xtables_error(PARAMETER_PROBLEM,
                                   "Need TCP, UDP, SCTP or DCCP with port specification");
                parse_ports(cb->arg, mr);
                break;
        case O_RANDOM:
                mr->range[0].flags |=  NF_NAT_RANGE_PROTO_RANDOM;
                break;
        }
}

```

上面的內容細節不太需要知道，只要知道其運作流程就是解析規則的參數並且設定相關的資料結構 **nf_nat_ipv4_multi_range_compat**，其中最重要的就是 **nf_nat_ipv4_multi_range_compat** 這個結構會需要定義兩份，一份給 **iptables** 的指令用，另外一份給 **kernel-space** 使用。

下面是該結構的定義，基本上對於 **MASQUERADE** 來說其兩個參數最主要設定的內容就是如何選擇 **source port**，當封包進行 **source NAT** 轉換後連接埠該怎麼選擇。
```c++
#ifndef _NETFILTER_NF_NAT_H
#define _NETFILTER_NF_NAT_H

#include <linux/netfilter.h>
#include <linux/netfilter/nf_conntrack_tuple_common.h>

#define NF_NAT_RANGE_MAP_IPS                    (1 << 0)
#define NF_NAT_RANGE_PROTO_SPECIFIED            (1 << 1)
#define NF_NAT_RANGE_PROTO_RANDOM               (1 << 2)
#define NF_NAT_RANGE_PERSISTENT                 (1 << 3)
#define NF_NAT_RANGE_PROTO_RANDOM_FULLY         (1 << 4)

#define NF_NAT_RANGE_PROTO_RANDOM_ALL           \
        (NF_NAT_RANGE_PROTO_RANDOM | NF_NAT_RANGE_PROTO_RANDOM_FULLY)

struct nf_nat_ipv4_range {
        unsigned int                    flags;
        __be32                          min_ip;
        __be32                          max_ip;
        union nf_conntrack_man_proto    min;
        union nf_conntrack_man_proto    max;
};

struct nf_nat_ipv4_multi_range_compat {
        unsigned int                    rangesize;
        struct nf_nat_ipv4_range        range[1];
};

struct nf_nat_range {
        unsigned int                    flags;
        union nf_inet_addr              min_addr;
        union nf_inet_addr              max_addr;
        union nf_conntrack_man_proto    min_proto;
        union nf_conntrack_man_proto    max_proto;
};

#endif /* _NETFILTER_NF_NAT_H */
```


## Kernel Space

原始碼的部分請參閱此連結 [Linux Kernel 4.15.18](https://elixir.bootlin.com/linux/v4.15.18/source
)

**kernel space** 則是整個封包邏輯運作的核心，包含了
1. 怎麼根據使用輸入的參數來挑選 **source port**
2. 如何動態選擇網路卡介面上的 **IP** 地址，如果有多個會怎麼選擇


我們的進入點是這個檔案 [ipt_MASQUERADE.c](https://elixir.bootlin.com/linux/v4.15.18/source/net/ipv4/netfilter/ipt_MASQUERADE.c#L34)

### 多重 IP 選擇
```
...
static unsigned int
masquerade_tg(struct sk_buff *skb, const struct xt_action_param *par)
{
	struct nf_nat_range range;
	const struct nf_nat_ipv4_multi_range_compat *mr;

	mr = par->targinfo;
	range.flags = mr->range[0].flags;
	range.min_proto = mr->range[0].min;
	range.max_proto = mr->range[0].max;

	return nf_nat_masquerade_ipv4(skb, xt_hooknum(par), &range,
				      xt_out(par));
}
...
```

概念也是非常簡單，從 **xt_action_param** 這個變數取出使用者對應的參數，並且往後呼叫其他參數去處理。

這邊可以看到其背後也是將 **xt_action_param** 裡面的**void*** 參數轉向 **nf_nat_ipv4_multi_range_compat** 這個物件來處理。

因為這些程式碼都是屬於 **kernel** 內的一部份，因此 **kernel** 也有一份跟上述非常相似的結構定義檔案，如下。

```
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _NETFILTER_NF_NAT_H
#define _NETFILTER_NF_NAT_H

#include <linux/netfilter.h>
#include <linux/netfilter/nf_conntrack_tuple_common.h>

#define NF_NAT_RANGE_MAP_IPS			(1 << 0)
#define NF_NAT_RANGE_PROTO_SPECIFIED		(1 << 1)
#define NF_NAT_RANGE_PROTO_RANDOM		(1 << 2)
#define NF_NAT_RANGE_PERSISTENT			(1 << 3)
#define NF_NAT_RANGE_PROTO_RANDOM_FULLY		(1 << 4)

#define NF_NAT_RANGE_PROTO_RANDOM_ALL		\
	(NF_NAT_RANGE_PROTO_RANDOM | NF_NAT_RANGE_PROTO_RANDOM_FULLY)

#define NF_NAT_RANGE_MASK					\
	(NF_NAT_RANGE_MAP_IPS | NF_NAT_RANGE_PROTO_SPECIFIED |	\
	 NF_NAT_RANGE_PROTO_RANDOM | NF_NAT_RANGE_PERSISTENT |	\
	 NF_NAT_RANGE_PROTO_RANDOM_FULLY)

struct nf_nat_ipv4_range {
	unsigned int			flags;
	__be32				min_ip;
	__be32				max_ip;
	union nf_conntrack_man_proto	min;
	union nf_conntrack_man_proto	max;
};

struct nf_nat_ipv4_multi_range_compat {
	unsigned int			rangesize;
	struct nf_nat_ipv4_range	range[1];
};

struct nf_nat_range {
	unsigned int			flags;
	union nf_inet_addr		min_addr;
	union nf_inet_addr		max_addr;
	union nf_conntrack_man_proto	min_proto;
	union nf_conntrack_man_proto	max_proto;
};

#endif /* _NETFILTER_NF_NAT_H */
```

接下來看看真正的邏輯處理函式 **nf_nat_masquerade_ipv4**

```c++
unsigned int
nf_nat_masquerade_ipv4(struct sk_buff *skb, unsigned int hooknum,
		       const struct nf_nat_range *range,
		       const struct net_device *out)
{
...
	rt = skb_rtable(skb);
	nh = rt_nexthop(rt, ip_hdr(skb)->daddr);
	newsrc = inet_select_addr(out, nh, RT_SCOPE_UNIVERSE);
	if (!newsrc) {
		pr_info("%s ate my IP address\n", out->name);
		return NF_DROP;
	}
    
	nat = nf_ct_nat_ext_add(ct);
	if (nat)
		nat->masq_index = out->ifindex;

	/* Transfer from original range. */
	memset(&newrange.min_addr, 0, sizeof(newrange.min_addr));
	memset(&newrange.max_addr, 0, sizeof(newrange.max_addr));
	newrange.flags       = range->flags | NF_NAT_RANGE_MAP_IPS;
	newrange.min_addr.ip = newsrc;
	newrange.max_addr.ip = newsrc;
	newrange.min_proto   = range->min_proto;
	newrange.max_proto   = range->max_proto;

	/* Hand modified range to generic setup. */
	return nf_nat_setup_info(ct, &newrange, NF_NAT_MANIP_SRC);
}
```

前面三個函式就是我們要關注的第一個重點，選出一個可用的地址作為封包之後的來源 **IP** 地址
```c++
	rt = skb_rtable(skb);
	nh = rt_nexthop(rt, ip_hdr(skb)->daddr);
	newsrc = inet_select_addr(out, nh, RT_SCOPE_UNIVERSE);
```
1. 從 **skb** 取出該封包對應的 `routing entry`
2. 從該 `routing entry` 中根據封包的目的位置選出下一個節點**next_hop**
3. 接下來根據兩個參數 **下一個節點的IP地址**, **已經決定好的輸出網路介面** 來決定最後使用的 **IP**

```c++
__be32 inet_select_addr(const struct net_device *dev, __be32 dst, int scope)
{
        __be32 addr = 0;
        struct in_device *in_dev;
        struct net *net = dev_net(dev);
        int master_idx;

        rcu_read_lock();
        in_dev = __in_dev_get_rcu(dev);
        if (!in_dev)
                goto no_in_dev;

        for_primary_ifa(in_dev) {
                if (ifa->ifa_scope > scope)
                        continue;
                if (!dst || inet_ifa_match(dst, ifa)) {
                        addr = ifa->ifa_local;
                        break;
                }
                if (!addr)
                        addr = ifa->ifa_local;
        } endfor_ifa(in_dev);

        if (addr)
                goto out_unlock;
no_in_dev:
        master_idx = l3mdev_master_ifindex_rcu(dev);
....
```

仔細觀察上述的流程之前，我們先來看一個擁有多個 **IP** 地址的範例
```bash=
vagrant@linux-study:~/linux$ ip addr show dev docker0
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:5a:4d:94:32 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet 172.19.1.3/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet 172.18.0.2/16 scope global secondary docker0
       valid_lft forever preferred_lft forever
    inet 172.18.0.3/16 scope global secondary docker0
       valid_lft forever preferred_lft forever
    inet 172.18.1.3/16 scope global secondary docker0
       valid_lft forever preferred_lft forever
```


這個範例中，我們針對 **docker0** 這個網卡設定了五組 **IP** 地址，其中根據網段分成兩大類
1. 172.18.0.0/16
   - 172.18.0.1/16
   - 172.18.0.2/16
   - 172.18.0.3/16
   - 172.18.1.3/16
3. 172.19.0.0/16
   - 172.19.1.3

第一個網段中總共有四個 **IP** 地址，上面的順序就是加入到系統的順序。可以觀察到這四個 **IP** 地址實際上系統中顯示的內容有些許不同
```bash=
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet 172.18.0.2/16 scope global secondary docker0
       valid_lft forever preferred_lft forever
    inet 172.18.0.3/16 scope global secondary docker0
       valid_lft forever preferred_lft forever
    inet 172.18.1.3/16 scope global secondary docker0
       valid_lft forever preferred_lft forever
```

可以觀察到除了第一個 **IP** 地址外，都被加上了 **secondary** 的描述詞，這邊可以猜想對於每個同網段的 **IP** 地址來說，由於都同網段，所以只會有一個主要的 **IP** 地址使用，而其他的都是作為 **secondary**。

第二個網段由於只有一個 **IP** 地址，本身也自然就沒有 **secondary** 的描述。

此外每個 **IP** 地址後面都有一個有趣的欄位 **scope**，這個欄位目前有四個值，分別如下，等等會再解釋這個值的用途

- global - the address is globally valid.

- site - (IPv6 only, deprecated) the address is site local, i.e. it is valid inside this site.

- link - the address is link local, i.e. it is valid only on this device.

- host - the address is valid only inside this host.

接下來我們再次認真的回歸到程式碼的層級去看一下邏輯

```c++
...
#define for_primary_ifa(in_dev)	{ struct in_ifaddr *ifa; \
  for (ifa = (in_dev)->ifa_list; ifa && !(ifa->ifa_flags&IFA_F_SECONDARY); ifa = ifa->ifa_next)
#define endfor_ifa(in_dev) }
 .... 
        for_primary_ifa(in_dev) {
                if (ifa->ifa_scope > scope)
                        continue;
                if (!dst || inet_ifa_match(dst, ifa)) {
                        addr = ifa->ifa_local;
                        break;
                }
                if (!addr)
                        addr = ifa->ifa_local;
        } endfor_ifa(in_dev);
...
```

把上述的 MACRO 給展開後會得到下列的迴圈內容

```c++
{ 
struct in_ifaddr *ifa; 
for (ifa = (in_dev)->ifa_list; ifa && !(ifa->ifa_flags&IFA_F_SECONDARY); ifa = ifa->ifa_next)
    if (ifa->ifa_scope > scope)
        continue;
    if (!dst || inet_ifa_match(dst, ifa)) {
        addr = ifa->ifa_local;
        break;
    }
    if (!addr)
        addr = ifa->ifa_local;
}
```

首先可以看到，該邏輯會透過迴圈的方式去取出 **(in_dev)->ifa_list** 內的所有清單，其實就是所有的 **IP** 地址，而終止條件條件很間單，遇到最後一個或是遇到第一個被標為 **secondary** 的 **IP** 地址就會停下來

接下來會開始比較每張網卡的 **scope** 是否跟參數的輸入值有關，這個範例中使用到的是 **global(RT_SCOPE_UNIVERSE=0)**，而上述的範例中全部都是 **global**，所以基本上可以忽略這個選項

接下來最重要的比較邏輯是 **如果知道目標位置，則會確認該目標位置與該目標**IP**的所屬網段是否符合**，是的話就可以用這個 **IP** 地址當作來源 **IP** 送出去。

```c++
static __inline__ bool inet_ifa_match(__be32 addr, struct in_ifaddr *ifa)
{
        return !((addr^ifa->ifa_address)&ifa->ifa_mask);
}
```
這個比較的概念非常簡單，首先全部的 **IP** 地址都適用 **32**位元的方式表達，為了判斷一個 **IP** 地址是不是屬於一個 **IP** 網段內的合法 **IP** 地址，其思考邏輯如下
假設地址 A = a.b.c.d, 網段則是 x.y.z.v/n
只要判斷 n bit 以前是否一樣即可，舉例來說 A = 172.17.8.23 , B = 172.17.12.53/16
最簡單的方式就是將 **IP** 的部分都用 **32** 位元表示，接者比較前 **n** 個位元是否完全一致即可
- A = 0xAC110817 => 10101100 00010001 00001000 00010111
- B = 0xAC110C35 => 10101100 00010001 00001100 00110101
- N = 16 => 11111111 11111111 00000000 00000000

所以先針對 **A/B** 兩個 **IP** 地址使用 **XOR** 的方式來找出差異點
- A^B => 00000000 00000000 00000100 00100010

接者針對前 N(16) 個位元比較
- (A^B)&N => 00000000 00000000 00000000 00000000

所以只要出來的結果是0 就代表前 (N) 個位元都一致，也可以表示 **A IP** 屬於 **B 網段**。


所以上面的 **IP**挑選邏輯可以簡單整理成

1. 針對所有非 **SECONDARY** 的 **IP** 地址依序檢查
2. 確認 **scope** 是否符合
    3. 確認該 **IP** 的網段是否包含 **目標IP地址**, 此範例中的 **目標IP地址** 則是根據路由表的 **next hop** 地址。


### 來源連接埠的選擇

接下來要看看到底怎麼選擇 **來源連接埠 (Source Port)**，這部分的選擇牽扯到太多的相關背景，譬如 **connection track**，甚至是 **DNAT** 的資料共用，所以在探索上會著重於 **SNAT** 相關的邏輯。


我們回到最初的函式 **nf_nat_masquerade_ipv4** 來看一下後面的處理

```c++
	/* Transfer from original range. */
	memset(&newrange.min_addr, 0, sizeof(newrange.min_addr));
	memset(&newrange.max_addr, 0, sizeof(newrange.max_addr));
	newrange.flags       = range->flags | NF_NAT_RANGE_MAP_IPS;
	newrange.min_addr.ip = newsrc;
	newrange.max_addr.ip = newsrc;
	newrange.min_proto   = range->min_proto;
	newrange.max_proto   = range->max_proto;

	/* Hand modified range to generic setup. */
	return nf_nat_setup_info(ct, &newrange, NF_NAT_MANIP_SRC);
```

這邊有三個欄位值得注意，分別是
1. newrange.flags
2. newrange.min_proto
3. newrange.max_proto

如果今天使用者在操作 **MASQUERADE** 的時候有設定其他參數 **--random**, **--to-ports** 的話，上述的欄位會有不同的數值，接下來的邏輯處理則會根據這些數值來處理。
- --random
    - flags = NF_NAT_RANGE_PROTO_RANDOM
- --to-ports
    - min_proto:max_proto => 使用者設定的 Port Number 區間
    - flags = NF_NAT_RANGE_PROTO_SPECIFIED


接下來我們就來看看 `nf_nat_setup_info` 這個函式裡面會怎麼做
```c++
unsigned int
nf_nat_setup_info(struct nf_conn *ct,
                  const struct nf_nat_range *range,
                  enum nf_nat_manip_type maniptype)
{
        struct net *net = nf_ct_net(ct);
        struct nf_conntrack_tuple curr_tuple, new_tuple;
...        
   
        get_unique_tuple(&new_tuple, &curr_tuple, range, ct, maniptype);
....        
}
```        

對於 **linux kernel** 來說，進行一個有效的 **NAT** 除了尋找到一個合法且唯一的**連接埠/IP 地址** 之外，如何讓這個連線能夠更有效的處理也是一個議題，總不可能每次該連線中的封包都要重新檢查，尋找並且轉換，這部分就會仰賴 **Conntrack (Connection Tracking)** 的整合與運作，能夠提供更快速的運作同時也可以避免 **NAT** 相關的規則每個封包都要執行一次。
因此上述的原始碼中會有非常大量的部分都跟 **Conntrack** 有關，這邊就不詳細談這概念，主要專注於 **NAT** 連接埠的選擇。

這邊我們專注於 **get_unique_tuple** 這個函式，這邊有五個參數，其中有兩個需要注意，分別是 **range** 以及 **mainiptype**。
其中 **range** 則是 **nf_nat_masquerade_ipv4** 根據使用者設定產生並傳入的，而 **mainiptype** 則是 **nf_nat_masquerade_ipv4** 傳入的 **NF_NAT_MANIP_SRC**.

```c++
/* Manipulate the tuple into the range given. For NF_INET_POST_ROUTING,
 * we change the source to map into the range. For NF_INET_PRE_ROUTING
 * and NF_INET_LOCAL_OUT, we change the destination to map into the
 * range. It might not be possible to get a unique tuple, but we try.
 * At worst (or if we race), we will end up with a final duplicate in
 * __ip_conntrack_confirm and drop the packet. */
static void
get_unique_tuple(struct nf_conntrack_tuple *tuple,
                 const struct nf_conntrack_tuple *orig_tuple,
                 const struct nf_nat_range *range,
                 struct nf_conn *ct,
                 enum nf_nat_manip_type maniptype)
{


...
        rcu_read_lock();
        l3proto = __nf_nat_l3proto_find(orig_tuple->src.l3num);
        l4proto = __nf_nat_l4proto_find(orig_tuple->src.l3num,
                                        orig_tuple->dst.protonum);
...                                        

        /* Last change: get protocol to try to obtain unique tuple. */
        l4proto->unique_tuple(l3proto, tuple, range, maniptype, ct);
}

```

從註解可以看到該函式的目的，因為 **MASQUERADE** 本身必須是 **POST-ROUTING** 位置才可以執行的 **TARGET**，所以也可以看到註解中有特別提到 **NF_INTER_POST_ROUTING** 的情況下，會做的就是改變 **source** 來源端相關的數值。

**NAT** 本身還有一個很有趣的概念就是針對不同的 **Layer4** 協定會有不同的選擇方式跟處理方式，目前總共有九種不同的實作，分別是

1. gre
2. icmp
3. icmpv6
4. dccp
5. sctp
6. tcp
7. udplite
8. udp
9. unknown

而我們的範例中專注於 **TCP** 本身的處理，所以接下來我們來看看 **TCP** 裡面的 **unique_tuple** 會怎麼處理
```c++
const struct nf_nat_l4proto nf_nat_l4proto_tcp = {
        .l4proto                = IPPROTO_TCP,
        .manip_pkt              = tcp_manip_pkt,
        .in_range               = nf_nat_l4proto_in_range,
        .unique_tuple           = tcp_unique_tuple,
#if IS_ENABLED(CONFIG_NF_CT_NETLINK)
        .nlattr_to_range        = nf_nat_l4proto_nlattr_to_range,
#endif
}; 
```

上述只是一個 TCP 結構的表達，實際上會有九種相關的物件都使用 **nf_nat_l4proto** 來註冊。

```c++
static void
tcp_unique_tuple(const struct nf_nat_l3proto *l3proto,
                 struct nf_conntrack_tuple *tuple,
                 const struct nf_nat_range *range,
                 enum nf_nat_manip_type maniptype,
                 const struct nf_conn *ct)
{
        nf_nat_l4proto_unique_tuple(l3proto, tuple, range, maniptype, ct,
                                    &tcp_port_rover);
}

void nf_nat_l4proto_unique_tuple(const struct nf_nat_l3proto *l3proto,
                                 struct nf_conntrack_tuple *tuple,
                                 const struct nf_nat_range *range,
                                 enum nf_nat_manip_type maniptype,
                                 const struct nf_conn *ct,
                                 u16 *rover)
{
        unsigned int range_size, min, i;
        __be16 *portptr;
        u_int16_t off;

        if (maniptype == NF_NAT_MANIP_SRC)
                portptr = &tuple->src.u.all;
        else
                portptr = &tuple->dst.u.all;

        /* If no range specified... */
        if (!(range->flags & NF_NAT_RANGE_PROTO_SPECIFIED)) {
                /* If it's dst rewrite, can't change port */
                if (maniptype == NF_NAT_MANIP_DST)
                        return;

                if (ntohs(*portptr) < 1024) {
                        /* Loose convention: >> 512 is credential passing */
                        if (ntohs(*portptr) < 512) {
                                min = 1;
                                range_size = 511 - min + 1;
                        } else {
                                min = 600;
                                range_size = 1023 - min + 1;
                        }
                } else {
                        min = 1024;
                        range_size = 65535 - 1024 + 1;
                }
        } else {
                min = ntohs(range->min_proto.all);
                range_size = ntohs(range->max_proto.all) - min + 1;
        }
        
        
        if (range->flags & NF_NAT_RANGE_PROTO_RANDOM) {
                off = l3proto->secure_port(tuple, maniptype == NF_NAT_MANIP_SRC
                                                  ? tuple->dst.u.all
                                                  : tuple->src.u.all);
        } else if (range->flags & NF_NAT_RANGE_PROTO_RANDOM_FULLY) {
                off = prandom_u32();
        } else {
                off = *rover;
        }

        for (i = 0; ; ++off) {
                *portptr = htons(min + off % range_size);
                if (++i != range_size && nf_nat_used_tuple(tuple, ct))
                        continue;
                if (!(range->flags & NF_NAT_RANGE_PROTO_RANDOM_ALL))
                        *rover = off;
                return;
        }
}
```

找 **Port** 連接埠的概念非常簡單
1. 先擬定一個範圍區間
2. 接下來於這個區間內找到一個起始位置，
3. 於這個位置嘗試，如果該 **port(連接埠)** 與 **source IP** 產生的 **tuple** 是唯一的，就可以使用，否則就遞加該 **port(連接埠)** (++).

```c++
        for (i = 0; ; ++off) {
                *portptr = htons(min + off % range_size);
                if (++i != range_size && nf_nat_used_tuple(tuple, ct))
                        continue;
                if (!(range->flags & NF_NAT_RANGE_PROTO_RANDOM_ALL))
                        *rover = off;
                return;
        }
```

上述的概念用程式碼表示就是，透過 **off** 絕對一個 **起始位置** 的偏移量    
透過 **min** 以及 **range_size** 決定範圍區間
最後透過 `++i` 以及 `nf_nat_used_tuple` 來遞加選擇的 **port** 並且確認是否為一。

看完了選擇 **port(連接埠)** 的邏輯後，我們就可以很清楚地猜到，使用者輸入的那些參數 **--to-ports**, **--random** 其實就是控制上面那些變量的初始值。

```c++
...
        if (!(range->flags & NF_NAT_RANGE_PROTO_SPECIFIED)) {
....
                } else {
                        min = 1024;
                        range_size = 65535 - 1024 + 1;
                }
        } else {
                min = ntohs(range->min_proto.all);
                range_size = ntohs(range->max_proto.all) - min + 1;
        }
...        
```
可以看到如果有設定 **NF_NAT_RANGE_PROTO_SPECIFIED** 這個參數的話，就會透過之前設定的 **min_proto/max_proto** 來決定 **min/range_size** 的大小。
否則一般情況下就是使用 **min=1024, range_size=65535-1024+1**.


```c++
        if (range->flags & NF_NAT_RANGE_PROTO_RANDOM) {
                off = l3proto->secure_port(tuple, maniptype == NF_NAT_MANIP_SRC
                                                  ? tuple->dst.u.all
                                                  : tuple->src.u.all);
        } else if (range->flags & NF_NAT_RANGE_PROTO_RANDOM_FULLY) {
                off = prandom_u32();
        } else {
                off = *rover;
        }
```

針對 **RANDOM** 的話，根據不同的用法有兩種方式去產生 **off**，如果都沒有特別指定的話就會使用 **rover** 這個全域變數來決定。
特別注意的是 **rover** 是一個 **static** 的變數，所以可以想成當前系統內的所有 **TCP 相關的NAT** 會透過一個共同的變數來決定當前要決定的起始區間。

## Summary
本篇文章透過閱讀原始碼的方式來學習 **MASQUERADE** 的運作方式，主要著重於兩個部分
1. 如何挑選 **IP** 地址，如果本身網路卡有多個 **IP** 地址的話，會怎麼挑選
2. 如何挑選 **source port(連接埠)**, 不同的參數會怎麼影響 **source port** 的選擇。

針對第一個問題，我們可以知道 **Linux Kernel** 本身對於相同網段的多重 **IP** 地址會有特別處理，除了第一個被設定的 **IP** 之外，其餘都會被設定為 **SECONDARY**，而選擇 **IP** 地址時則會依序詢問每個 **非 SECONDARY** 的 **IP** 地址，並且確認目標 **IP** 地址是否屬於選到的 **IP** 地址。

針對第二個問題，我們觀察到 **MASQUERADE** 有提供額外兩個參數，分別是 **--to-ports** 以及 **--random**。這兩個變數最後會影響怎麼挑選
1. **port** 的可用區間
2. 起始 **port** 的偏移量

下篇文章我們會嘗試透過直接修改 **source code** 的方式來觀察整個問題並驗證上述的觀察結果。
