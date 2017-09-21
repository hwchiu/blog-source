---
title: OVS + DPDK + Docker 共同玩耍
keywords: 'SDN,Network,Linux,Ubuntu,DPDK,OVS'
date: 2017-09-15 13:33:17
tags:
	- SDN
	- Ubuntu
	- Linux
	- Kernel
	- Network
	- DPDK
	- OpenvSwitch
description:
---

最近與 [John Lin](https://www.facebook.com/linton.tw) 一起合作 OpenvSwitch 時遇到了一些問題，由於此問題實在過於有趣，所以決定寫下這篇文章來記錄此問題。

此問題會牽扯到三個元件，分別是 `OVS`, `DPDK` 以及 `Docker`，首先來看一下網路拓墣，如下。

![](https://i.imgur.com/X8Rriqu.jpg)

<!--more-->

首先，準備好兩台機器，其中一台機器為 **Linux Based** 機器，在其環境中安裝了
1. OpenvSwitch 2.8
2. Docker + ubuntu image
3. NIC (Support DPDK)
4. DPDK 17.05.2

此環境中，我們透過 `ovs-vsctl` 創造了一個 `userspace mode` 的 `ovs switch(datapath)`, 其命名為 `ovs_br`。
然後我們透過 `docker` 與 [pipework](https://github.com/jpetazzo/pipework) 在系統上開啟了兩個基本的 `ubuntu`   container, 並且將這兩個 container 都掛到 `ovs_br` 上。
這兩個 `container` 裡面都有一張網卡 **eth0**、ip 地址分別是 10.55.66.8 以及 10.55.66.7。
然後將 **ovs_br** 設定為 10.55.66.1
最後再將系統上的網卡以`DPDK`的形式連接到 **ovs_br** 上。

接下來我們透過一般的 switch 將該 `Linux Based` 的機器與一般的 server相連，該 server 的IP 設定為 `10.55.66.10`。

整個網路環境就上圖所呈現。

所以目前網路中有四個元件可以用來進行操作，分別是兩台 Container， Linux Host 本體，以及外面的那台 server。

首先我們使用`ping` 指令來測試網路狀況，沒意外的任意兩台機器都能夠順利的連接到對方。然而接下來使用**TCP**作為測試時，卻發現了一些詭異的情況。

在上述四種狀況中，總共有六種組合可以測試，然而其中卻只有一種組合能夠讓 TCP/UDP 順利連接成功。此組合就是 **Linux Host** 配上 **外面那台 server**。
其他組合完全沒有辦法建立起一條 **TCP Connection**。

為了解決此問題，我們使用了下列方式釐清問題所在
1. tcpdump 擷取封包，以 TCP 為例，發現只要是 **Container** 內部傳送回來的封包，雖然透過 `tcpdump` 可以看到該封包回來，但是另外一端的應用程式都不會將該封包收上去處理。因此對於 **TCP** 來說沒有辦法建立起一個正常的連線。
2. 懷疑是 **container** 的問題，所以手動用 **ip netns** 創建了兩個 **namespace**, 結果有一樣的問題。
3. 懷疑是 **OVS** 的問題，因此將 **OVS** 切換成 **Linux bridge**(DPDK也取消使用)，結果問題就順利解決了，一切網路都通了。
4. 懷疑是 **Userspace OVS** 的問題，因此切換成 **kernel module mode**(DPDK取消)，結果問題也是順利解決了。

最後經過網路上不停的搜查，我們找到了一篇文章在講類似的問題
[connection-issue-between-docker-container-and-other-machine](https://stackoverflow.com/questions/45167203/connection-issue-between-docker-container-and-other-machine)

最後嘗試透過該文章所提到的方法，我們到 **Container** 裡面透過`ethtool`將其對外網卡`eth0`相關的 `TX/RX offloading` 功能關閉，果然功能就一切正常了。

為了釐清這個這個功能為什麼會對網路造成這些影響，我們重新利用`tcpdump`擷取封包，並且使用 `wireshark` 來觀察，發現了一個有趣的事情。

以下好所有圖示中的ＩＰ位置可能與上述拓墣不同，不過重點不在那邊，所以忽略即可。

首先，我們可以看到當網路不通時 **wireshark** 的解析，如下圖
![](https://i.imgur.com/uQ4LKoJ.png)
可以看到對於 **TCP** 連線來說，其實只有 **SYN** 以及 **SYN/ACK**，主要是 **Client** 端沒有將該 **SYN/ACK** 給收起來，最後導致一連串的重送。
所以接下來針對 **SYN** 以及 **SYN/ACK** 兩個封包去看看。

![](https://i.imgur.com/VxcKkWo.png)
![](https://i.imgur.com/26wuweJ.png)

這兩個封包最有趣的地方就是，其 **TCP** 標頭檔怎麼看都不一樣，結果 **Checksum** 卻完全一樣，這邊看起來就是有問題。馬上懷疑是這邊造成收端沒有辦法收起對應的 **SYN/ACK**。
於是接下來馬上去試試看正常的 **TCP** 連線。

![](https://i.imgur.com/Ue15uf7.png)
![](https://i.imgur.com/fYe9AT9.png)
![](https://i.imgur.com/72or38f.png)

結果如同預期般，不同封包因為 **TCP** 標頭檔不同，其 **Checksum** 也不同。
所以整個問題就是 **Container** 內的 **TX/RX offload** 造成了其 **Checksum** 出現問題導致無法讓封包正確被處理。
而且此問題只會出現在 **OVS+DPDK** 上。

所以我又跑回去翻了一下最初 OVS+DPDK 相關的程式碼，在最初版本的 `netdev-dpdk.c` 中有提到
```c++
+Restrictions:
 +-------------
 +
 +  - This Support is for Physical NIC. I have tested with Intel NIC only.
 +  - vswitchd userspace datapath does affine polling thread but it is
 +    assumed that devices are on numa node 0. Therefore if device is
 +    attached to non zero numa node switching performance would be
 +    suboptimal.
 +  - There are fixed number of polling thread and fixed number of per
 +    device queues configured.
 +  - Work with 1500 MTU, needs few changes in DPDK lib to fix this issue.
 +  - Currently DPDK port does not make use any offload functionality.

```

其中的 `Currently DPDK port does not make use any offload ` 其中的段話讓我滿好奇的，但是在最新 OVS 2.8 中該敘述也已經不見了, 可能此限制也已經排除。所以我們為什麼會遇到這個問題，暫時還沒有頭緒，等有時間時再來細追看看，不然就先去 **ovs-dicuss** 那邊發問一下好了。
最後感謝 [John Lin](https://www.facebook.com/linton.tw) 大力幫忙，一起研究這個問題，讓彼此都學到了一些鬼玩意。

Reference
- [First release of netdev-dpdk](https://github.com/openvswitch/ovs/commit/8a9562d21a40c765a8ae6775a070cb279cb2147a#diff-c43dadca1fdb46e2bf2e3f928a8529fbR77)
-  [connection-issue-between-docker-container-and-other-machine](https://stackoverflow.com/questions/45167203/connection-issue-between-docker-container-and-other-machine)
