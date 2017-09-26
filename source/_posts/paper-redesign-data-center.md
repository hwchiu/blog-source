---
title: "[論文導讀] Re-architecting datacenter networks and stacks for low latency and high performance"
keywords: 'SDN,Network,Linux,Ubuntu'
date: 2017-09-26 13:19:37
tags:
	- SDN
	- Network
	- DPDK
	- P4
	- System
description:
---

IGCOMM 2017**, 可以在[這邊](http://dl.acm.org/citation.cfm?id=3098825)找到網路上的文章。於 **SIGCOMM 2017**中，其所屬於的 section 是 **Programmable Devices**，所以可以料想到本文內容會跟 **Programmable Devices** 有些關係。果不其然的內容中有提到 **P4**, **NetFPGA**，處此之外，本篇內容也提到了 **DPDK**, **Clos Network**, **MPTCP**, **DCTCP**, **incast** 等，必須說本篇文章包含了內容實在廣泛，所以要花不少時間去釐清每個元件，才可以了解到本篇論文的內容。

由於本篇論文長達**14**頁, 因此這邊導讀的文章本身也不會太簡短，所以一開始就先針對本文做一個簡單的介紹，直接瞭解到本論文希望做什麼，達到什麼目的，為什麼要這樣做。

本論文所發生的場景是在 **Data Center**內，不適用於一般的網際網路，主要是因為網際網路充滿太多未知性與無法掌控的裝置，譬如每條 Link 的頻寬, Switch 的設定等。有這些未知的情況下，無法設計一個好的運作模式來滿足 **Low Latency** 與 **High Throughut** 的需求。因此環境都只考慮 **Data Center**。

本論文提出了 NDP (New Datacenter Protocol Architecture) 這個概念，其具體目標，評比對象以及實作方法如下。

本文希望能夠達到的需求有
1. 對於 **short-flow** 能夠有盡量低的 **latency**
2. 對於 **long-flow** 能夠有盡量高的 **throughput**
2. 對於一個高度負載的網路中，能夠充分利用整個網路拓墣上的 Link 來傳輸封包 (high network capacity)
3. 能夠將 **incast** 情況對網路造成的影響降到最小

本文評比對象有
1. **DCTCP** (Data Center TCP)
2. **MPTCP** (Multi Path TCP)
3. **DCQCN** (Data Center QCN)
> DCQCN is a congestion control protocol for large scale RDMA networks, developed jointly by Microsoft and Mellanox.

本文實作方法
1. 透過 **DPDK** 打造 NDP Application (拋棄 TCP, 重新打造 L4 Protocol)
2. 於 **NetFPGA SUME**上面實作 NDP 相關功能，作為一個 **Hardware Switch**
3. 於 **P4**上面實作 NDP 相關功能，作為一個 **Software Switch**


看到這邊如果對於上述的內容感到興趣的話，就可以開始往下看了!
<!--more-->

## Abstract
近年來 **Data Center** 蓬勃發展，為了能夠在內部提供更良好的網路效能，不論是低延遲或是高產出，網路架構從早期的三層式架構(**Fat tree**)逐漸都轉換成 [Clos Network](https://www.networkworld.com/article/2226122/cisco-subnet/clos-networks--what-s-old-is-new-again.html)，然而傳統的 **TCP** 協定在設計上並不是針對 **Data Cetner**來設計的，因此其設計原理導致其不能滿足需求。
因此本論文提出了 NDP (new data-center transport architecture)，其能夠提供 **short transfer**接近完美的傳輸時間，同時對於廣泛的應用情況，如 **incast** 下亦能夠提供很高的傳輸效能。
NDP架構下， **switch** 採用非常小的 **buffer**來存放封包，當 **buffer** 滿載時，不同於傳統的將封包丟掉，NDP採取的是截斷封包的內容(payload)，只保留該封包的標頭檔，並且將此封包設定為高優度優先轉送。這種作法讓收端能夠有更完整關於送端的資訊，能夠動態的調整傳送速率。
本論文將 NDP 的概念實作於 **Software switch**, **Hardware Switch** 以及一般的 **NDP application**。這中間使用的技術與硬體分別包含了 **DPDK**、**P4** 以及 **NetFPGA**。
最後本論文進行了一系列效能評比，於大規模的模擬中證明了NDP能夠為整個 **data center** 提供低延遲與高輸出的特性。

## Introduction
隨者 **Data Center** 爆炸性的成長，為了滿足其內部傳輸的需求(低延遲/高輸出)，有各式各樣的新方法或是舊有技術的改進用來滿足上述需求。譬如 TCP 改進的 DCTPC、本來就存在已久的 MPTCP 甚至是 RDMA (RoCE)。 若對 **RDMA/RoCE** 想要更瞭解，可以參考下列連結。[RDMA Introduction (一)](http://hwchiu.com/rdma-introduction-i.html)
再 **RDMA** 的網路環境中，為了減少封包遺失對整體傳輸造成的傳輸，都會希望能夠將整個網路傳輸打造成 **lossless** 的環境，為了達成這個方法，可以採用 **Ethernet Flow Control**， **Explicit Congestion Notification**(ECN) 或是 **Priority Flow Control**(PFC)。然而在 ** SIGCOMM 2016** 微軟發了一篇 paper 再講 RDMA + PFC 的問題。其表明雖然 PFC 可以用來控制傳送封包的速率，藉此達到 **lossless** 的網路環境，但是一旦整體網路處於高度壅塞的情況時，資料封包與 **PFC** 控制封包的爭奪會使得整體網路沒有辦法繼續提供**低延遲**的優點，最後給出了一個結論 *“how to achieve low network latency and high network throughput at the same time for RDMA is still an open problem.“*

在這篇論文內，我們提出了一個不同與以往思維的新協定 NDP 來滿足上述的要求。
NDP 不同於 **TCP**，不需要先透過三方交握來進行連線才可以傳輸封包，同時也避免掉 **TCP** **slow start** 的機制，可以連線一開始就採用全速的方式去傳送封包。
此外， NDP 採用了 **per-packet multipath** 而不是 **per-connection multipath** 的方式同時透過 NDP 協定的方式避免掉同條連線中封包順序亂掉(Out of order)所產生的問題。
對於 **NDP switch**來說，採用了類似 **CP(Cut Payload)** 的機制，當**switch** 的緩衝區滿載的情況下，對於接下來收到的封包不會直接丟掉，而是會將其 **Payload** 給移除，並且設定高優先度的方式將其標頭檔盡可能快速的傳送到目的地，讓接收段能夠有資訊瞭解到當前網路與發送端的狀態。
透過 **CP** 的機制可以對 **metadata** 達到近乎 **lossless** 的狀況，不過對於 **data** 來說則沒有辦法。不過因為 **NDP** 本身協定設計的方式，封包掉落所產生的影響並不會像 **TCP** 一樣有如此嚴重的影響。

## Design Space
對於內部使用的 **Data center** 來說，其網路流量大部分都是屬於 **Request/Reply**,這種類似 RPC 方式的流量。這意味者以網路使用率來看其實不會一直處在滿載的狀況，但是對連線接收端的應用程式(譬如 server)來說，其所收處理的網路流量可能是大資料的傳輸，也可能是簡單資料的短暫連線。對於短暫連線來說，最大的重點就是**低延遲**，盡可能快的傳輸回去。
為了處理這個問題，今日有滿多的應用程式決定採取 **reuse** TCP 連線來處理多個 request，藉此方式減少 **TCP** 三方交握所產生的延遲。
然而，是否存在一種架構，不論當前整個網路系統是否處於高流量負載，該架構能夠讓應用程式不需要重複使用連線，讓每個 **request** 都是一條全新連線，同時又能夠接近完美的延遲(意味者不會像 **TCP** 都要先經過三方交握才可以開始傳資料)。
於是乎 **NDP** 的設計發想就出現了，為了達到上述的要求，首先必須要重新思考，當前網路架構中，是什麼因素阻礙了上述要求，這些阻礙要如何克服。
因此接下來的章節會講述這些因素。

### End-to-end Service Demans
本文列出了四個 **data center** 內應用程式要追求的特性。
#### Location Independence
一個分散式的應用程式其運行於**data center**內的任一機器上都不應該要影響應用程式本身的功能。由於目前都採用 **clos network** 的方式來設計網路拓樸，所以整個網路拓樸方面應該是能夠提供足夠的流量來使用，而不會變成一個理由或是瓶頸使得應用程式必須要選擇特定的機器才可以運行。
#### Low Latency
雖然 **Clos Network** 的架構能夠提供足夠的頻寬給拓墣中內的服務器，對於流量的負載平衡能提供良好的效果，但是對於短暫連線來說，並沒有辦法提供低延遲的能力。
作者認為雖然能夠提供高流量輸出是一個很重要的議題，但是能夠提供低延遲的能力則是更重要且優先度更高。
#### Incast
**Incast** 這個專有名詞的介紹可以參考[Data Centers and TCP Incast - Georgia Tech - Network Congestion](https://www.youtube.com/watch?v=-e5Rw2I3QJk)，簡單來說就是同時間有大量的**request**進到**data center**內，這些**request**都會產生對應的**reply**然後這些**reply**連線同時間一起進入到 **switch**內，這會使得 **switch** 的緩衝區立刻滿載沒辦法繼續承受封包，導致部分的 **TCP** 連線需要降速重送。
作者認為一個好的網路架構遇到這種問題時，應該要能夠保護背後的應用程式連線，讓這些連線應該要盡可能地維持低延遲性。
#### Priority
這邊特性老實說有點難以想像，我盡力的就我所瞭解的去解釋。
假設今天有一個應用程式，同時會發送不同的 **request** 到後端去處理，而這些 **request** 所產生的 **reply** 本身是有依賴的關係。
所以假如這多個 **reply** 沒有依照當初發送 **request** 的順序回來的話，在應用程式端這邊就必須要特別去處理，譬如說不要同時送多個**request**，不過這樣就沒有更好的效能表現。
因此對於應用程式來說，若本身能夠有一個優先度的機制，能夠決定那些封包要先處理，就可以解決上述的問題，而讓應用程式本身就可以更自由的去處理。

### Transport Protocol
目前 **data center** 內部採用的 **Transport Protocol** 雖然可以處理上述應用程式的問題，但是為了處理那些問題，反而會失去下列特性。
而 NDP 本身在設計時，希望能夠在滿足上述應用程式的需求，同時又保有下列的特性。

#### Zero-RTT connection setup
目前主流的 **TCP** 協定再傳送資料前，必須要先進行一次三項交握，這意味者當 **data** 送出去前，至少要先花費 **RTT*3** 的等待時間。
對於低延遲的應用程式來說，希望能夠達到 **RTT*0**，至少 **RTT*1** 的等待時間就能夠將資料送出，這意味者對 **NDP** 來說，在資料發送前，整個連線不需要有三方交握類似的行為，才可以一開始就直接送出資料。

#### Fast start
**Data Center** 不同於網際網路的地方在於網路拓樸中的每個 **Switch/Link** 都是自己掌握的。
所以 **TCP** 採用的 **Slow Start** 其實對於 **Data Center** 來說是沒有效率的，畢竟一開始就可以知道可用頻寬多少，不需要如同面對網際網路般的悲觀，慢慢地調整 **Window Size**，而是可以一開始就樂觀的傳送最大單位，在根據狀況進行微調。
若採取這種機制，則應用程式可以使用更快的速度去傳送封包。

#### Per-packet ECMP
在 **Clos Network** 中，錯綜複雜的連結狀態提供了 **Per-flow** ECMP 一個很好發揮的場所，可以讓不同的連線走不同的路徑，藉此提高整體網路使用率。
然而如果今天 **Hash** 的結果相同的話，其實是有機會讓不同連線走相同的路徑，即使其他路徑當前都是閒置的。
若採取 **MPTCP** 這種變化型的 **TCP** 來處理的話，該協定本身的設計可以解決上述的問題，但是對於短暫流量或是延遲性來說，卻沒有很好的效果。
為了解決這個問題，如果可以將 **Per-flow** ECMP 轉換成 **Per-Packet** ECMP。因此 NDP 本身協定的設計就是以 **Per-Packet** ECMP 為主。

#### Reorder-tolerant handshake
假設我們已經擁有了 **Zero RTT** 以及 **Per-packet ECMP** 兩種特性，擇一條新連線的封包可能就會發生 **Out of order** 的情況，收送順序不同的狀況下，若對於 **TCP** 來說，就會觸發壅塞控制的機制進而導致降速。
因此 NDP 在設計時，必須要能夠處理這種狀況，可以在不依賴封包到達先後順序下去處理。

#### Optimized for Incast
即使整個 **data center** 的網路環境，如頻寬等資訊一開始就已經可以掌握， **incast** 的問題還是難以處理，畢竟網路中變化太多，也許有某些應用程式就突然同時大量產生封包，這些封包同時間到達 **switch** 就可能導致封包被丟棄。
因此 NDP 本身在設計時，也希望能夠解決這個問題。

