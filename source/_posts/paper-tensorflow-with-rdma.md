---
title: '[論文導讀] - Towards Zero Copy Dataflows using RDMA'
keywords: 'AI,Network,Linux,Kernel,RDMA,Tensorflow'
tags:
  - AI
  - Network
  - RDMA
  - TensorFlow
  - Linux
  - Paper
abbrlink: 24428
date: 2018-01-23 02:27:30
description:
---

Introduction
============

2017 SICCOM 上面出現了一篇令人感到有趣的論文，內容是 **Towards Zero Copy Dataflows using RDMA**，可以到 [這邊](https://dl.acm.org/citation.cfm?id=3131975) 閱讀該篇論文內容。

再看一篇論文前，最重要的就是其摘要，根據其摘要我們可以快速的對該篇論文有一個基本的認識，所以先來看一下該篇論文的摘要

# ABSTRACT
>Remote Direct Memory Access (RDMA) offers ultra-low latency and
CPU bypass networking to application programmers. Existing applications
are often designed around socket based software stack that
manages application buffers separately from networking buffers
and do memory copies between them when sending/receiving data.

先大概介紹了一下 RDMA 的特性與價值，想要瞭解更多關於 RDMA 的可以參考[這篇文章](https://www.hwchiu.com/rdma-introduction-i.html)。

<!--more-->

>With large sized (up to hundreds MB) application buffers, the cost of
such copies adds non trivial overhead to the end-to-end communication
pipeline.

這篇就是問題的重點，大檔案資料在複製過程中產生的負擔，也是本篇論文想要解決的問題。

>In this work, we made an attempt to design a zero copy
transport for distribute dataflow frameworks that unifies application
and networking buffer management and completely eliminates
unnecessary memory copies.

這裡提到，這篇論文怎麼解決上述問題，而且直接瞭當的說明其方法就是，**zero copy transport for distrubute dataflow frameworks**.

> Our prototype on top of TensorFlow
shows 2.43x performance improvement over gRPC based transport
and 1.21x performance improvement over an alternative RDMA
transport with private buffers and memory copies.


最後說明上述的 **distrubute dataflow frameworks** 採用的是 **tensorflow**，並且評比了一下效能 ((job 完成的時間))。
這邊比較的是原生的 gRPC 版本，以及 **Yahoo** 所開發的版本
其中 Yahoo　開發的版本可以在 [Github](https://github.com/yahoo/TensorFlowOnSpark ) 這邊看到


該篇論文內容滿簡短的，只有短短的三頁，大抵上分成
1. 有什麼問題
2. 這個問題可以怎麼解決
3. 效能評估

詳細的實作內容跟架構設計都用用短短的文字帶過而已，所以在看這篇論文的時候，必須要有一定的背景知識，才能夠更容易地去想像作者想要表達的意思。

Tensorflow
==========
由於這篇文章著使用的 **tensorflow**,並且是基於分散式運算的 **tensorflow**，所以在開始本文之前，必須要先來複習一下分散式的 **tensorflow**架構，並且了解一下其基本的運作行為與概念。
這邊就不介紹，可以直接參考 [TensorFlow 基本使用與分散式概念](https://kairen.github.io/2017/04/10/tensorflow/intro/) 這篇文章來學習

# INTRODUCTION
近年來，不論是小/中/大企業，或是資料中心內部的網路交換機都轉換成10G/40G，甚至是100G，愈來愈多高速網路的需求。再軟體方面，為了能夠達到這些高速網路(高流量，低延遲性)，有很多相關的技術都被發展出來，譬如 Intel 的 **DPDK**， Mellanox 推出的 **ASAP2** 以及本文的主角 **RDMA(Remote Directly Memory Access)**。
然而世界上大部分運行於 **Linux** 上的網路應用程式，幾乎都是採用 **BSD** 規範的 **SocketAPI** 來進行網路程式開發，這意味者
1. 所有的網路封包都是依賴 **Linux Kernel** 的 **TCP/IP** 來進行處理
2. 每個從應用程式送出去的封包都要再 **User-Space** 與 **Kernel-Space** 傳遞，每次的傳遞都牽扯到資料的交換。
然而上述提到的相關技術，本身於程式撰寫方面都會有特別的 **API** 要使用，因為大部分的概念都是趨近於 **Kernel Bypass**，整個封包的流程都會跳過 **Kernel**，所以沒有辦法直接使用傳統 **BSD** 的 **API**。
以 **RDMA** 來說，為了讓現有的應用程式能夠簡單的直接轉移到 **RDMA** 的使用，有了如 **RSocket** 或 **SDP** 這類設計來模擬傳統 **BSD API** 接口的 **API**。
根據本文作者的描述，這類型的 **API** 並沒有辦法完全的使用到 **RDMA** 的優勢，還是有一些額外的負擔產生使得封包延遲性提高。

除了網路之外，再 **AI**發展方面，**Dataflow** 這種架構在近年的軟體中可是蓬勃發展並被使用，不論是 **Spark**, **Naiad**, **Hadoop** 以及 **Tensorflow** 這些框架內都有 **Dataflow** 的概念存在。
**Dataflow** 本身是一個有向無迴圈的圖，如下圖
![](https://i.imgur.com/IYecwlJ.png)
其中我們稱沒有任何輸入的點叫做 **Source**, 而沒有任何輸出的點稱為 **Sink**

在該圖中，每一個節點都意味者一個操作/運算，而每個節點的輸出都會是下一個節點的輸入，所以再分散式的架構中，這些圖就會構建成整個資料傳輸的路徑。
根據作者的觀察，再一些典型的使用情境中，這些輸入/輸出的資料都是一些大型且不會變動的資料。
而且根據上述所言，這些資料在當前的網路應用程式設計中，都會牽扯到多次的複製。而這些多次的複製行為就會產生負擔，對於整個傳輸行為造成不小的影響。

所以是否能夠透過上述的技術，透過 **Kernel Bypass** 或是 **Zero Copy** 等概念來解決上述問題，並且提升整體的傳輸效率就是本文的想要解決問題的方法。

在本文中，作者使用了 **RDMA** 的方式來完成的**點對點**之間的 **Zero Copy**，其實作的重點如下
1. 基於 Tensorflow
2. 設計一款新的 **Memory Allocator**，能夠解析當前分散式架構中的圖表資訊，並且收集緩衝區使用的相關資訊。藉由這個辦法與 **RDMA** 的整合，移除系統中沒必要的記憶體複製行為
3. 根據效能評測 (job 完成的時間)，若比較對象為原生基於 TCP/IP 與 gRPC 的 **Tensorflow**，其效能能夠提升 2.43 倍，所比較對象是 **YAHOO** 所開發的 **Tensorflow**，其效能也能夠提升 1.21 倍。


# DESIGN
## System & Library
目前現存的 **dataflow** 相關應用程式再底層傳輸資料時，大抵上分成兩種方式傳輸，一種是採用 **RPC** 的方式去設計收送的資料格式，另外一種則是基於檔案區塊(File Block)進行傳輸。
舉例來說， **Tensorflow** 以及早期版本的 **Spark** 就是使用 **RPC** 的方式進行跨節點的資料傳輸。後期版本的 **Spark** 以及 **Hadoop (MapReduces)** 則是將檔案輸出到本地的檔案系統，然後再將該檔案區塊傳輸到其他節點。
作者認為，**RPC** 於傳輸有者效能上的缺失，主要歸咎於 **RPC** 的請求/回復(Request/Response) 本身不但有大量的資料複製，同時也有可能有加解密(Encoding/Decoding)。 
這些行為都會嚴重的消耗 **CPU**運算，並且降低點到點的傳輸速度與提高延遲性。
至於檔案區塊(File Block)通常都會將檔案存到外部的檔案系統，即使是 **RAMFS(Memory Based File System)** 也是會牽扯到很多資料複製的行為。

作者認為上述的問題主要在於**API**的設計行為，此行為導致計算(**computation**)與傳輸(**communication**)兩個子系統都自行管理自己的記憶體空間，彼此沒有共享。
這種情況下很難真正去實現點對點的 **Zero Copy** 的資料傳輸，(這邊也呼應前面的 rSOCKET的設計依然會有非必要的資料複製行為)。
作者認為在這種架構下，不論這兩個子系統都設計了多良好的緩衝區管理(Buffer Management)，都至少會有一次的資料複製行為。
舉例來說，假如我們採用了最廣為流行的 **TCP/IP** 系統，在 **Linux Kernel** 內部會採用 **sk_buff** 的結構來儲存網路封包，而且在底層大部分針對網路封包操作的行為都是基於指標去操作，雖然這類看起來都是指標操作的行為的確避免了任何資料複製的動作。
但是資料要從 **User Space** 轉送到 **Kernel Space** 還是需要經過一次的轉換。
這類型的問題不論是在各種 **RPC** 或是 函式庫(**Library**)內都有出現。

## Dataflow
對於 **Dataflow** 來說，作者認為若在高速網路環境中同時滿足下列兩種情形，則記憶體複製就有機會是整體的效能瓶頸。
1. 作者認為若應用程式的緩衝區太大，沒有辦法符合當前系統架構上的 **L1/L2/L3** 快取。
2. 很少有單一應用程式有辦法把整個網路環境中的頻寬都吃滿，造成壅塞。

首先，針對第一個情形來說，我們要先探討複製大檔案跟複製很多小檔案的差距。
作者認為當經過一些運算時候，後者(很多小檔案)會更有機會留在快取中繼續使用，而大檔案很容易就會發生 **Cache-Miss**的事件而最終要從 **Memory** 中讀取。
作者也提到，對很多 **RDMA** 相關的應用程式(譬如 **FaSST** )來說，針對小檔案的檔案複製，會特別處理。該應用程式內會特別準備一塊空間來處理這些小檔案的複製，而這塊空間是個 **page-locked** 的記憶體空間。藉由這個行為減少每次對於小檔案都要去進行所謂的 **pinning memroy**(從虛擬記憶體空間映射到實際記憶體空間)產生的消耗。
接下來探討第二個情形，有一些應用程式(譬如 **KV** )是特別關注在封包延遲性方面的，這種應用程式通常都不會消耗整個網路頻寬，其要求的點都是在於**每秒能夠處理多少封包**，而大部分情況下，**Key**/**Value**這些資料都是小封包，大概是數**KB**左右。
此外，作者還提供了一個數據來強調這個情況。
當傳輸的緩衝區大小是 **4KB** 左右的，這時候大概傳輸可以達到 **20-30 GB/s**，而若緩衝區的大小是超過 **4MB** 時，這時候的傳輸只剩下 **2-4 GB/s**,主要是緩衝區太大的時候，這些封包都沒有辦法符合快取的大小，導致大量複製行為最終使得傳輸速度下降。
所以結論一下，只要當緩窗區的大小過大的時候，這時候很容易因為 **Cache Miss**而產生各種複製的行為，而這些複製行為就會導致整體效能下降。

為了解決這個問題，作者提出了一個 **Unified Memory Alloccator** 的機制，這個機制會去同時控管計算(**computation**)與傳輸(**communication**)兩個子系統內的緩衝區配置。
這個機制有兩大重點
1. 實作不同類型的記憶體配置，譬如當前記憶體是要給 **RDMA** 使用，還是當前本系統的 **DMA**。
2. 解析 **dataflow graph** 中的資訊來決定當前節點的資料是要使用何種記憶體配置。

此外，不論是 **RDMA** 或是 **DMA** 所產生的緩衝區都會同時在計算(**computation**)與傳輸(**communication**)兩個子系統內共同使用，直到兩個系統都不再使用該空間時，才會將該空間給釋放出來。

所以看到這邊可以大概想像一下作者到底要怎麼做了。
分析整個 **dataflow graph** 中的資料走向，盡可能的讓相同使用的資料只要使用一份緩衝區空間即可，然後透過 **RDMA** 或是 **DMA** 等技術來傳遞資料，減少整個過程中的資料複製行為。

# IMPLEMENTATION
作者採用的 **dataflow** 是基於 **tensorflow**，因此該 **Unified Memory Alloccator** 本身也是實作在 **tensorflow** 裡面，可以直接到[下列位置](https://github.com/tensorflow/tensorflow/pull/11392)觀看作者與 **tenforflow** 維護者的溝通以及程式碼的修改。
作者新增了一種 **memory allocator** 到整體的程式碼內，要使用時只要打開相關選項即可(前提是要先針對有 **RDMA** 重新編譯整個專案)。

在作者的實作的記憶體分配器中，會自動的去解析 **tensorflow** 的 **computational grpah** 以及 **distributed graph partition**，所以只有滿足下列兩種條件的張量 **tensor** 才會去該記憶體分配器中被選擇使用 **RDMA**。
- 必須是source node(出發端點) 或是 sink node(結尾端點)
- 該操作必須是 send/receive 同時要跨實體機器
這也很合理，因為跨機器間的傳輸會用到網路，所以才會需要用到 **RDMA** 來傳輸資料。

在最原始的 **TenforFlow** 的版本中，使用了基於 **HTTP/2** 的 **gRPC** 格式作為 **Tensor** 間的傳輸。而作者修改了這邊的程式碼，使得這邊會直接跳過 **RPC** 的呼叫，改使用 **RDMA** 來傳輸資料。
此外，為了完整支援能夠透過 **RDMA** 來使用本地端或是遠端的 **GPU** 資源，作者還使用了 **GPU direct RDMA** 的技術來完成這些事情。藉由這個技術任何 **PCI-e** 的第三方裝置(譬如 **NIC** )都可以直接讀取 **GPU** 上面的記憶體空間。
此外作者也觀察到當 **GPU direct RDMA** 的路徑會經過 **CPU-socket** (譬如處於不同的 **NUMA NODE**) 會發生嚴重的效能問題。
因為在這種情況下，這些資料會先被送到記憶體內，然後在被複製出來，最後才會真正的送到 **NIC** 或是 **GPU** 上去處理，導致了不必要的消耗。

簡單來說，作者透過分析然後決定那些記憶體空間要共用，那些用**DMA**，那些用**RDMA**，同時透過 **GPU direct RDMA** 的技術直接存取遠方機器上面 **GPU** 的記憶體，藉此降低大量的資料複製行為。


# EVALUATION
### Environment (Hardware)
1. 4 servers 都連接上 Mellanox MSN2100-BB2F 40Gbe RoCE Switch (For RDMA)
2. 每個 server 都配置下列硬體
    - Mellanox MT27500 40GbE NIC
    - Dual6-core Intel Xeon E5-2603v4 CPU
    - 4 NVidia Tesla K40m GPUs
    - 256 GB DDR4-2400MHz
3. Switch 本身有配置 **PFC** 來控管傳輸流量降低封包掉落機率

### Environment (Software)
1. 作者訓練了分散式版本的 **VGG16** CNN 模型。
2. 模組的參數大小是 528 MB
3. 採用了 **synchronous** 模式

**parameter servers** 的數量跟 **Workers** 的數量一至，同時
針對每一台機器上面的 **Worker** 都會同時使用 **CPU** 與 **GPU** 來進行運算，而 **parameter server (PS)** 則只會使用 **CPU** 配上系統上的記憶體來收集資訊。

### Target
1. 作者修改後的 **TensorFlow**
2. 官方未修改的 **TensorFlow** 
    - v1.2.1
3. Yahoo 自行修改後的 **TensorFlow**
    - 也支援 RDMA，但是機制沒有作者完善，還是有不少的複製行為

### Metric
作者想觀察的是完成訓練所消耗的時間，

### Result
1. 比較的結果來看，(1) 比 (2) 快上了 **2.43** 倍, 而 (1) 比 (3) 快上了 **1.21** 倍
2. 16顆 **GPU** 與 1顆 **GPU** 的比較起來，效能提升了 **13.8**倍


# Summary
作者觀察到在 **dataflow** 中之間的傳輸大小不小，同時這些資料會在系統中有大量的資料複製行為，因此引進了 **DMA**，**RDMA** 以及 **GPU direct RDMA** 等技術來減少整體的資料複製行為，並且也將整體的程式碼完全貢獻回 **TensorFlow** 內，未來任何人想要嘗試這個機制也可以直接使用。

