---
title: 'Linux NAT Masquerade 研究(下)'
keywords: 'Network,Linux,Ubuntu,iptables,nat,snat,masquerade'
abbrlink: 32301
date: 2020-01-02 05:38:43
tags:
  - Network
  - Linux
  - Iptables
description: 本篇文章透過修改 MASQUERADE Kernel Module 原始碼的方式來觀察系統的變化，專注於當 NAT 功能執行前後， conntrack 這個結構的改變。透過不同的實驗與環境來觀察 1)多重 IP 的情況下怎選擇 2)指定不同參數時，連接埠的變化。此外為了簡化整體操作過程，將整個實驗環境都透過 Vagrant 打包成為一個可重複執行的環境，並且也準備好可以編譯 Kernel Module 的環境與指令。
---

# Preface
本篇文章銜接上篇 [Linux NAT Masquerade 研究(上)](https://www.hwchiu.com/iptables-masquerade.html)，透過不同的方式來觀察 **MASQUERADE** 的運作結果。

相較於上篇文章是透過觀察原始碼的方式來學習過程，本篇文章嘗試於直接修改相關程式碼的方式來觀察最後選擇的結果

為了希望讓整個測試環境是簡單且可重複執行的，所以決定使用 **Vagrant** 來包裝整個環境，並且限制使用的軟體版本，譬如 **Ubuntu**, **Linux Kernel** 等

此外由於 **MASQUERADE** 本身使用到的函數牽扯到 **Kernel Module** 以及 **Linux Kernel**，所以在建置環境的時候其實有點麻煩。
對於 **Kernel Module** 來說相對簡單，只要準備好相關的環境支援編譯，最後重新安裝相關的 **Kernel Module** 即可。
但是整個 **Linux Kernel** 想到還要全部編譯並且安裝就覺得麻煩，所以後來決定單純針對 **Kernel Module** 的部分去修改，因此能夠觀察到的部分也會被受限，沒有辦法太細部的針對前文的每個函式去觀察。

# 環境設定
本篇文章的環境可透過 **Vagrant** 進行設定，相關內容都存放於此 [hwchiu/network-study](https://github.com/hwchiu/network-study/tree/master/iptables/masquerade) 專案內。

- Ubuntu: 18.04
- Linux Kernel: 4.15

## 設定環境
```bash
git clone https://github.com/hwchiu/network-study
pushd network-study/iptables/masquerade
vagrant up
```

透過上述方式即可進入到測試環境中

# 觀察實作

這次的觀察著重於 **Kernel Module** 的部分，所以主要是針對 `ipt_MASQUERADE.c` 這個檔案，針對這檔案我進行了一些修改，主要是增加一個輸出相關資訊的函式，並且根據 **NAT** 呼叫前後來呼叫該函式輸出

## 修改程式碼
```c++
 static unsigned int
 masquerade_tg(struct sk_buff *skb, const struct xt_action_param *par)
 {
+       unsigned int ret =0;
        struct nf_nat_range range;
        const struct nf_nat_ipv4_multi_range_compat *mr;

@@ -71,13 +70,8 @@ masquerade_tg(struct sk_buff *skb, const struct xt_action_param *par)
        range.min_proto = mr->range[0].min;
        range.max_proto = mr->range[0].max;

+       printk("before nat\n");
+       show_status(skb, par);
+       ret = nf_nat_masquerade_ipv4(skb, xt_hooknum(par), &range,
-       return nf_nat_masquerade_ipv4(skb, xt_hooknum(par), &range,
                                      xt_out(par));
+       printk("after nat\n");
+       show_status(skb, par);
+       return ret;
 }
```

最主要的修改就是執行 **nf_nat_masquerade_ipv4** 前後去呼叫 **show_status** 來輸出相關資訊。

## Conntrack

往下繼續講之前，先稍微簡單提一下 **Conntrack** 的概念，對於每一條**網路連線**, Linux Kernel 都會用 **Conntrack** 這個結構去紀錄該連線的資訊。
每一條網路連線都代表者雙向的傳輸方向，這邊不管是 TCP/UDP 都享有這類型結構的紀錄，以下圖為例

每個 **節點** 都會有一份屬於自己的 **conntrack** 紀錄，而每個 **conntrack** 都會有**Original**,**Reply** 兩個方向的紀錄。

最簡單的情況下， **Original** 以及 **Reply** 完全顛倒的，就是 **Original** 的來源端相同於 **Reply** 的目的端。

![](https://i.imgur.com/ei2eOSL.png)

## 結果觀察

```c++

static void show_status(struct sk_buff *skb, const struct xt_action_param *par)
{
	struct nf_conn *ct;
	struct nf_conntrack_tuple *tuple;
	enum ip_conntrack_info ctinfo;

	ct = nf_ct_get(skb, &ctinfo);
	tuple = &(ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple);
	printk("ORIGINAL: outgoing interface: %s, src-ipv4:%pI4, src-port:%u, dst-ipv4:%pI4, dst-port:%u\n", xt_out(par)->name, (void*)&tuple->src.u3.ip, ntohs(tuple->src.u.tcp.port),(void*)&tuple->dst.u3.ip, ntohs(tuple->dst.u.tcp.port));

	tuple = &(ct->tuplehash[IP_CT_DIR_REPLY].tuple);
	printk("REPLY: outgoing interface: %s, src-ipv4:%pI4, src-port:%u, dst-ipv4:%pI4, dst-port:%u\n", xt_out(par)->name, (void*)&tuple->src.u3.ip, ntohs(tuple->src.u.tcp.port),(void*)&tuple->dst.u3.ip, ntohs(tuple->dst.u.tcp.port));
	return;
}
```

而 **show_status** 本身則是會去輸出，**conntrack** 的兩個方向，也就是所謂的 **ORIGINAL** 以及 **REPLY**。
輸出的內容概括了
1. 封包出去的網卡
2. 該 **conntrack** 記錄到的來源 **IP**以及來源連接埠
3. 該 **conntrack** 記錄到的目標 **IP**以及目標連接埠

所以我們真正觀察的內容就是
**執行 NAT 前後 Conntrack 的變化**


# 實驗

## 環境介紹
本文章的所有環境都已經透過 **Vagrant** 來建立一個獨立且可重複的環境，主要的[邏輯](https://github.com/hwchiu/network-study/blob/master/iptables/masquerade/Vagrantfile)是
1. 安裝 Linux Kernel header
2. 下載修改後的 kernel module 原始碼
3. 編譯該 Kernel module 並且安裝
4. 安裝 Docker 準備測試環境

```bash
    # Install modified kernel module
    git clone https://github.com/hwchiu/network-study
    sudo apt-get install -y linux-headers-$(uname -r)
    cd network-study/iptables/masquerade/module
    sudo make
    sudo cp ipt_MASQUERADE.ko /lib/modules/$(uname -r)/kernel/net/ipv4/ netfilter/ipt_MASQUERADE.ko

    # Install ntp
    sudo apt-get install -y ntp
    # Install Docker
    export DOCKER_VERSION="18.06.3~ce~3-0~ubuntu"
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install -y docker-ce=${DOCKER_VERSION}

```

事先準備好 **Vagrant** 的環境，可以用下列的指令進入到實驗環境

```bash
git clone https://github.com/hwchiu/network-study
pushd network-study/iptables/masquerade
vagrant up
vagrant ssh
```

## 連接埠觀察

第一個實驗則是觀察參數的運作，即 **--to-ports** 的運作方式即結果，為了完成這個測試，我們需要做相關的操作
1. 刪除 docker 所安裝的 iptables 指令
2. 安裝 kernel module (非必要，因為 vagrant 啟動時已經安裝到系統內)
3. 重新安裝 iptables 指令，並且設定 **--to-ports**
   - 這邊要注意的是 **--to-ports** 本身只有特定的 Layer4 協定有支援，但是 **DNS** 並不支援，所以需要額外額外一條 iptables 規則來處理 DNS 的查詢。
4. 透過 **docker run** 的方式去產生 **TCP** 封包，並且觀察 **kernel** 的輸出結果

### 設定 iptables
```bash=
pushd ~/network-study/iptables/masquerade/module/
sudo iptables -t nat -D POSTROUTING -s 172.18.0.0/16 ! -o docker0 -j MASQUERADE || true
sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 -p tcp ! -o docker0 -j MASQUERADE --to-ports 55666-55680|| true
sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o docker0 -j MASQUERADE || true
```
如果今天自己有任何修改 **kernel module** 需求，則需要採用下列方式
```bash=
pushd ~/network-study/iptables/masquerade/module/
sudo iptables -t nat -D POSTROUTING -s 172.18.0.0/16 ! -o docker0 -j MASQUERADE || true
make
sudo rmmod ipt_MASQUERADE
sudo insmod ipt_MASQUERADE.ko
sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 -p tcp ! -o docker0 -j MASQUERADE --to-ports 55666-55680|| true
sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o docker0 -j MASQUERADE || true
```
主要差異在於我們需要重新安裝建制後的 `kernel module` 到系統內

### 測試流量
由於我們是透過 **docker** 的環境來使用 **MASQUERADE**，所以隨便找一個可以對外連線的 **docker** 來測試即可

```bash=
sudo docker run --rm --entrypoint "/usr/bin/wget" hwchiu/netutils google.com
```

### 觀察結果

```bash=
sudo dmesg -c
```

輸入上述指令後，會看到類似於下列的輸出，我們挑重點來看就好
```bash=
[  371.727940] original
[  371.727943] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:33894, dst-ipv4:8.8.8.8, dst-port:53
[  371.727944] REPLY: outgoing interface: eth0, src-ipv4:8.8.8.8, src-port:53, dst-ipv4:172.18.0.2, dst-port:33894
[  371.727945] after nat
[  371.727946] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:33894, dst-ipv4:8.8.8.8, dst-port:53
[  371.727947] REPLY: outgoing interface: eth0, src-ipv4:8.8.8.8, src-port:53, dst-ipv4:10.0.2.15, dst-port:33894
[  371.754760] original
[  371.754763] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:37622, dst-ipv4:216.58.200.46, dst-port:80
[  371.754764] REPLY: outgoing interface: eth0, src-ipv4:216.58.200.46, src-port:80, dst-ipv4:172.18.0.2, dst-port:37622
[  371.754766] after nat
[  371.754767] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:37622, dst-ipv4:216.58.200.46, dst-port:80
[  371.754768] REPLY: outgoing interface: eth0, src-ipv4:216.58.200.46, src-port:80, dst-ipv4:10.0.2.15, dst-port:55666
[  371.792826] original
[  371.792851] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:42676, dst-ipv4:8.8.8.8, dst-port:53
[  371.792859] REPLY: outgoing interface: eth0, src-ipv4:8.8.8.8, src-port:53, dst-ipv4:172.18.0.2, dst-port:42676
[  371.792862] after nat
[  371.792863] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:42676, dst-ipv4:8.8.8.8, dst-port:53
[  371.792864] REPLY: outgoing interface: eth0, src-ipv4:8.8.8.8, src-port:53, dst-ipv4:10.0.2.15, dst-port:42676
[  371.807542] original
[  371.807545] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:34276, dst-ipv4:172.217.160.100, dst-port:80
[  371.807546] REPLY: outgoing interface: eth0, src-ipv4:172.217.160.100, src-port:80, dst-ipv4:172.18.0.2, dst-port:34276
[  371.807548] after nat
[  371.807549] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:34276, dst-ipv4:172.217.160.100, dst-port:80
[  371.807550] REPLY: outgoing interface: eth0, src-ipv4:172.217.160.100, src-port:80, dst-ipv4:10.0.2.15, dst-port:55666
[  371.982956] docker0: port 1(veth1cf6a5c) entered disabled state
[  371.984851] veth96adc5c: renamed from eth0
[  372.014944] docker0: port 1(veth1cf6a5c) entered disabled state
[  372.016375] device veth1cf6a5c left promiscuous mode
[  372.016378] docker0: port 1(veth1cf6a5c) entered disabled state
```

因為 **wget** 抓取網頁的過程中，包含了 **HTTP** 連線以及 **DNS** 查詢，所以會有很多條 **conntrack** 產生，不太意外，最後的 **docker** 則是因為 **docker --rm** 跑完就結束的關係，導致相關的 **veth** 被收回。

我們抓前兩條 **conntrack** 來看即可，這邊要注意我們的參數是 **--to-ports 55666-55680**

```bash=
[  371.727940] before nat
[  371.727943] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:33894,
                dst-ipv4:8.8.8.8, dst-port:53
[  371.727944] REPLY: outgoing interface: eth0, src-ipv4:8.8.8.8, src-port:53,
                dst-ipv4:172.18.0.2, dst-port:33894
[  371.727945] after nat
[  371.727946] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:33894,
                dst-ipv4:8.8.8.8, dst-port:53
[  371.727947] REPLY: outgoing interface: eth0, src-ipv4:8.8.8.8, src-port:53,
                dst-ipv4:10.0.2.15, dst-port:33894
```

針對第一條 **conntrack**，觀察如下

**Before NAT**
- ORIGINAL
    - source_ip: 172.18.0.2
    - source_port: 33894
    - destination_ip: 8.8.8.8
    - destination_port: 53
- REPLY
    - source_ip: 8.8.8.8
    - source_port: 53
    - destination_ip: 172.18.0.2
    - destination_port: 33894

預設情況下系統都會假設 **ORIGINAL** 跟 **REPLY** 是完全對稱的，畢竟這是最簡單的模式。

**After NAT**
- ORIGINAL
    - source_ip: 172.18.0.2
    - source_port: 33894
    - destination_ip: 8.8.8.8
    - destination_port: 53
- REPLY
    - source_ip: 8.8.8.8
    - source_port: 53
    - destination_ip: 10.0.2.15
    - destination_port: 33894

這邊可以看到一旦執行 **NAT** 後， **ORIGINAL** 完全沒有改變，但是 **REPLY** 裡面的 **destination_ip** 則更動了。
對於這樣的改動可以有下列的解讀方式
1. 預設情況下， **MASQUERADE** 對於 **DNS** 不會特別幫你挑選連接埠，而是你本來送過來前是什麼，就是什麼。 **TCP也是，可以自行修改規則去驗證**
2. 對於 **Linux Kernel** 來說，這條 **conntrack** 的兩個方向分別對應
   - 來源封包從容器送到網卡
   - 回應封包從外部到網卡
3. 當這條 **conntrack** 被確認之後，接下來所有從 **OREIGINAL** 的封包就會自動地被修正其 **source ip**，就不會被 **iptables** 的規則再次執行了，算是一個小加速。


```bash=
[  371.754760] before nat
[  371.754763] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:37622,
                dst-ipv4:216.58.200.46, dst-port:80
[  371.754764] REPLY: outgoing interface: eth0, src-ipv4:216.58.200.46, src-port:80,
                dst-ipv4:172.18.0.2, dst-port:37622
[  371.754766] after nat
[  371.754767] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:37622,
                dst-ipv4:216.58.200.46, dst-port:80
[  371.754768] REPLY: outgoing interface: eth0, src-ipv4:216.58.200.46, src-port:80,
                dst-ipv4:10.0.2.15, dst-port:55666
```

接下來看一下 **wget** 指令的主體， **HTTP* 連線的狀況

針對第一條 **conntrack**，觀察如下

**Before NAT**
- ORIGINAL
    - source_ip: 172.18.0.2
    - source_port: 37622
    - destination_ip: 216.58.200.46
    - destination_port: 80
- REPLY
    - source_ip: 216.58.200.46
    - source_port: 80
    - destination_ip: 172.18.0.2
    - destination_port: 37622

預設情況下系統都會假設 **ORIGINAL** 跟 **REPLY** 是完全對稱的，畢竟這是最簡單的模式。

**After NAT**
- ORIGINAL
    - source_ip: 172.18.0.2
    - source_port: 37622
    - destination_ip: 216.58.200.46
    - destination_port: 80
- REPLY
    - source_ip: 216.58.200.46
    - source_port: 53
    - destination_ip: 10.0.2.15
    - destination_port: 55666

這邊可以看到一旦執行 **NAT** 後， **ORIGINAL** 完全沒有改變，但是 **REPLY** 裡面的 **destination_ip** 以及 **destination_port** 則更動了。

這邊的解讀去上述相同，唯一不同的是 **port** 最後變成 **55666**，這個就是我們先前透過 **--to-ports 55666-55680** 去設定的區間。

## IP 選擇觀察

第二個實驗則是要觀察如果網卡有多重 **IP** 的話，會怎麼選擇，為了完成這個測試我們需要做相關的操作
1. 對主要輸出網卡新增一個不同網段的 **IP** 地址
2. 透過 **docker run** 的方式去產生 **TCP** 封包，並且觀察 **kernel** 的輸出結果

這部分我們可以直接沿用上個實驗的 **iptables** 規則，所以不需要額外處理，當然如果有需要修改 **kernel module** 來觀察的話，就需要進行一樣的動作

### 設定IP

Vagrant 的環境中，eth0 是主要的對外網卡，我們對其增加一個不同網段的 **IP**，同時也增加了一個路由規則，希望送往 **1.1.1.1** 的封包會嘗試先走到 **1.2.3.4** 去
```bash=
sudo ip addr add 1.2.3.56/24 dev eth0
sudo route add 1.1.1.1 gw 1.2.3.4 dev eth0
```

接者我們觀察預設的路由表與 IP 設定
```bash=
vagrant@linux-study:~/network-study/iptables/masquerade/module$ sudo route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    100    0        0 eth0
1.1.1.1         1.2.3.4         255.255.255.255 UGH   0      0        0 eth0
1.2.3.0         0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.0.2.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.0.2.2        0.0.0.0         255.255.255.255 UH    100    0        0 eth0
172.17.8.0      0.0.0.0         255.255.255.0   U     0      0        0 eth1
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0

vagrant@linux-study:~/network-study/iptables/masquerade/module$ ip addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:c2:be:11 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 81500sec preferred_lft 81500sec
    inet 1.2.3.56/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fec2:be11/64 scope link
       valid_lft forever preferred_lft forever
```

這邊只要確認
1. 1.1.1.1/32 會走 **eth0** 並且 gateway 是 1.2.3.4
2. 1.2.3.0/24 會走 **eth0**，因為 **eth0** 本身有相同網段的 **IP**，因此就不需要 **gateway** 的處理，直接轉發即可。


### 測試流量

```bash=
vagrant@linux-study:~/network-study/iptables/masquerade/module$ sudo docker run --rm --entrypoint "/usr/bin/wget" hwchiu/netutils 1.1.1.1
--2020-01-02 04:28:29--  http://1.1.1.1/
Connecting to 1.1.1.1:80... failed: No route to host.
```

這邊我們不需要考慮目的地的網頁伺服器是否真的存在，同時這個範例中也不考慮 **DNS** 的查詢，我們直接觀察 **TCP** 的結果即可


根據前篇文章的探討， **MASQUERADE** 會先查詢 **routing table** 中的規則得到要送出的 **next-hop**。接者從目標網卡的眾多 **非 SECONDARY** **IP** 地址中去挑選一個符合的來送出。

按照上述的猜想，整個過程會
1. 根據 routing table, `1.1.1.1` 的 `next-hop` 是 `1.2.3.4`
2. `eth0` 上面能夠符合 `1.2.3.4` 的 IP 是 `1.2.3.56/24`

所以最後預期送出去的 **IP** 必須是 **1.2.3.56


接下來觀察 `1.1.1.1` 的結果並驗證猜測
```bash=
[ 1879.823479] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:59734,
                dst-ipv4:1.1.1.1, dst-port:80
[ 1879.823480] REPLY: outgoing interface: eth0, src-ipv4:1.1.1.1, src-port:80,
                dst-ipv4:172.18.0.2, dst-port:59734
[ 1879.823482] after nat
[ 1879.823483] ORIGINAL: outgoing interface: eth0, src-ipv4:172.18.0.2, src-port:59734,
                dst-ipv4:1.1.1.1, dst-port:80
[ 1879.823484] REPLY: outgoing interface: eth0, src-ipv4:1.1.1.1, src-port:80,
                dst-ipv4:1.2.3.56, dst-port:55666
```

**Before NAT**
- ORIGINAL
    - source_ip: 172.18.0.2
    - source_port: 59734
    - destination_ip: 1.1.1.1
    - destination_port: 80
- REPLY
    - source_ip: 1.1.1.1
    - source_port: 80
    - destination_ip: 172.18.0.2
    - destination_port: 59734

預設情況下系統都會假設 **ORIGINAL** 跟 **REPLY** 是完全對稱的，畢竟這是最簡單的模式。

**After NAT**
- ORIGINAL
    - source_ip: 172.18.0.2
    - source_port: 59734
    - destination_ip: 1.1.1.1
    - destination_port: 80
- REPLY
    - source_ip: 1.1.1.1
    - source_port: 80
    - destination_ip: 1.2.3.56
    - destination_port: 55666

這邊可以觀察到最後轉換的 **IP** 地址則是 **1.2.3.56** 而不是最初 **eth0** 上面的 **10.0.2.15/24**, 也算是稍微驗證了一下多重 **IP** 的選擇，並非單純的先來後到，而是還要考慮到網段的符合。


# Summary
本篇文章透過修改 **MASQUERADE** 原始碼的方式來觀察其運作的方式，得到的結論是 不論是 **SNAT** 或是 **DNAT**，其最後的修改都體現在 **conntrack** 這個結構上，透過修改 **ORIGINAL** 或是 **REPLY** 兩個方向來預期最後看到的封包結構。

對於 **MASQUERADE(TCP/DNS)** 來說，預設的情況下不會幫你修改封包的來源連結埠，若有需要修改必須要透過之前提過的參數 **--to-ports** 以及 **--random** 來修改，本文中沒有特別提到 **--random** 的部分是因為與 **--to-ports** 大同小異，自行修改相關指令就可以進行測試。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
