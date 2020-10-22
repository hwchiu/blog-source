---
title: OVS + DPDK + Docker 共同玩耍(二)
keywords: 'OVS,dpdk, docker,container,veth'
tags:
  - SDN
  - Linux
  - Kernel
  - DPDK
  - OpenvSwitch
abbrlink: 8151
date: 2017-09-21 15:26:23
description: 本文延續前篇文章關於 Docker/OpenvSwitch/DPDK 整合遇到的連線問題，此文章會專注於這個連線問題，從問題發生的原因到如何解決，以及該問題為什麼會在上述的組合中發生都進行一些研究與分析，雖然最後還沒有找到真正造成封包損壞的原因，但是至少也把問題範圍給縮小到 OpenvSwitch/DPDK 上.

---

# Preface

此文章主要接續前篇文章 [OVS + DPDK + Docker 共同玩耍](https://www.hwchiu.com/ovs-dpdk-docker.html#more) 進行後續探討。

根據 [Jalen Lin](https://www.facebook.com/jalen.lin.10?fref=ufi&pnref=story) 提出的一篇文章 [Linux kernel bug delivers corrupt TCP/IP data to Mesos, Kubernetes, Docker containers](
https://tech.vijayp.ca/linux-kernel-bug-delivers-corrupt-tcp-ip-data-to-mesos-kubernetes-docker-containers-4986f88f7a19)，其中的原因似乎可以用來解釋此問題。

因此本篇文章會分成兩個部分，第一部份是先針對上述文章進行探討，第二部分則是將第一部分的結果與先前經驗去結合，來追出更深層的問題所在。

# Problems

[Linux kernel bug delivers corrupt TCP/IP data to Mesos, Kubernetes, Docker containers](
https://tech.vijayp.ca/linux-kernel-bug-delivers-corrupt-tcp-ip-data-to-mesos-kubernetes-docker-containers-4986f88f7a19) 該篇文章中提到他們使用 **docker** 配上 **veth** 一起使用時，會發現 **TCP** 的連線有機率會不通。
>**veth** 是用來將 **Docker/Container** 與 **Host** 本身串接的一種方法，每個 **veth** 都有兩個端點，從一端點進去的封包，就會從另外一個端點出來。因此透過這個技術就能夠讓封包在 **Host** 與 **Docker/Container** 之間傳遞。

詳細檢查分析後，發現當出現問題時，該 **TCP** 封包的 **Checksum** 是錯誤的，所以才會導致另外一端的應用程式沒有辦法收起該封包。
與我前述文章相同的是，只要將 **Docker**/**Container** 內的網卡相關的 **TX/RX offloading** 的功能關閉，則上述的問題就不會再出現了。

經過努力，他們最後將問題給縮小到 **Veth** 並且找出了 **Root Cause**。
原來是因為若網卡本身有設定 **TX/RX Offloadiong** 的設定下，封包經過 **veth** 時，當初為了速度的最佳化，這邊就會省略 **CHECKSUM** 的檢查。
這就意味者當若當時封包的 **Checksum** 有錯誤時，該封包的 **CehckSum** 不會被重新計算而就以這個錯誤的型態往外發送，導致收端看到的就是錯誤的 **CehckSum**。
一旦將 **TX/RX Offloading** 給關閉後，則 **Veth** 那邊就不會去處理 **CheckSum** 相關的邏輯，所以後續處理的部份就有機會將該 **CheckSum** 重新計算來校正該封包。

所以這邊條列一下整個問題發生的過程
1. 封包本身因為不知名原因損毀（文章提到這不是不可能的，畢竟每個封包都會經過大量的 hardware/software 來處理)
2. 封包到達 **Docker/Container** 內處理，回覆的封包透過 **veth** 時 **CheckSum** 相關的設定被設定為 **CHECKSUM_UNNECESSARY**
3. 搭配者 **CHECKSUM_UNNECESSARY** 且本身就有損毀的封包就這樣從 **Host** 送出去到達目的地，卻因為 **CheckSum** 不正確，所以沒有被收起來。

該團隊最後送了一條 **Patch** 到 **Linux Kernel** 去修正這個問題，修正的方法則是 **veth** 那邊不要去對 **Checksum** 的設定有任何更動

詳細的 **patch** 內容如下。

```c++
diff — git a/drivers/net/veth.c b/drivers/net/veth.c
index 0ef4a5a..ba21d07 100644
 — — a/drivers/net/veth.c
+++ b/drivers/net/veth.c
@@ -117,12 +117,6 @@ static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
 kfree_skb(skb);
 goto drop;
 }
- /* don’t change ip_summed == CHECKSUM_PARTIAL, as that
- * will cause bad checksum on forwarded packets
- */
- if (skb->ip_summed == CHECKSUM_NONE &&
- rcv->features & NETIF_F_RXCSUM)
- skb->ip_summed = CHECKSUM_UNNECESSARY;

 if (likely(dev_forward_skb(rcv, skb) == NET_RX_SUCCESS)) {
 struct pcpu_vstats *stats = this_cpu_ptr(dev->vstats);
```

# Study

再探討完畢 **veth** 的問題後，要如何與我之前的問題給整合？
首先, 根據上述文章的說明，該 **Patch** 只有 **backport** 回到 **Linux 3.14**, 而我的測試環境是 **Linux Kernel 3.10**，所以這意味者我的系統上並沒有上述的 **Patch**, 因此 **veth** 是有問題的。

確認 **veth** 有問題後，接下來就要確認為什麼 **TCP** 封包本身會有損毀，因為若沒有損毀的話，其實 **veth** 這邊的邏輯是不會造成封包有任何問題的。

根據我的測試結果，Linux Bridge/OVS Kernl Datapath/OVS Userspace Datapath 三種 `software switch` 中只有 **OVS Userspace** 會造成問題，因此我猜測是 **OVS Userspace** 會造成 **TCP** 封包的 **CheckSum** 出錯。

為了驗證我的想法，我繼續使用下圖的拓樸來進行驗證。
首先於圖中 **Container2** 架設一個簡易的 `www server`，然後使用右圖的機器作為一個 `www client`，當 `www client` 嘗試與 `www server` 建立TCP連線時，我於圖中標示**1**,**2**,**3** 三處分別使用 **Tcpdump** 去擷取封包來觀察 **TCP**封包的 **CheckSum** 是在哪邊出問題。
![](https://i.imgur.com/d15wsP9.jpg)

根據 **Tcpdump** 的結果，三處所看到的TCP回應封包(SYN/ACK)資訊如下
1. SYN/ACK 的 **CheckSum** 正確
2. SYN/ACK 的 **CheckSum** 正確
3. SYN/ACK 的 **CheckSum** 不正確，其數值與 SYN 封包相同

這實驗結果印證了我的猜測，封包在經過 **OVS userspace datapath** 後封包就出錯了，這就意味者 **OVS** 這邊的處理過程可能有問題。此外，值得注意的是 SYN/ACK 封包的 **CheckSum** 與 SYN 封包是一樣的，這邊更令人覺得應該是**OVS**處理上有錯誤。

總結來看，我遇到的問題有兩個
1. **veth** 的問題使得系統不會重新校正 **CheckSum**
2. **OVS** 的問題使得 TCP 封包出現錯誤

基本上到這邊已經大致上找出問題點了，最後一步驟就是翻進 **OVS** 的程式碼內，找出對應的錯誤，若沒有時間找出來，就發一個 issue 到官方去詢問好了。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/mLwKb3zrx

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4XNG

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