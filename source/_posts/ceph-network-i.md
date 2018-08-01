---
title: Ceph Network Architecture 研究(一)
tags:
  - Ceph
  - Network
  - SOS
  - ScaleOut
  - SDS
  - SourceCode
keywords: 'Ceph,Network,SourceCode,SDS,SOS'
abbrlink: 13656
date: 2017-05-23 08:12:45
---

此篇文章主要研究目前很熱門的 **scale out storage** 軟體 **ceph** 的網路架構。
由於 scale-out 的特性，不同節點彼此之間需要透過網路來交換資料，所以 **ceph**
這邊提供了 **public network** 以及 **cluster network** 兩種不同的用途的網路架構。

其中 **cluster network** 是用來傳輸資料用的，當各節點有資料要同步時，流量都是走這個網路來交換，
而 **public network**則是處理剩下的流量，譬如外面的資料讀取(read/write)等。

因此接下來將針對 **Ceph** 內部的網路架構進行研究，並且最後選擇一種類型進行更進一步的研究。
<!--more-->

### Environment
- 本文所觀看的架構以及程式碼都基於 [ceph 12.0.0](https://github.com/ceph/ceph/tree/v12.0.0)
- 所有網路的程式碼都放在 **/src/msg/** 內部資料夾內。
- 可以透過修改 **ceph.conf**中的 **ms_type** 來改變當前 ceph 要使用何種網路架構來連線。

### Introduction
在此版本中， **ceph** 內部的網路架構大致上可以分成三種來使用
- Simple
- Async
- Xio

若要更細微去分的話，則 **Async** 又可以細分成三成類型，分別是
- Posix
- DPDK
- RDMA

接下來會介紹一下這些類型的概念。

#### Simple
Simple顧名思義就是簡單，其網路架構也是非常的簡單，是 **ceph** 最早期的設計
每一對節點之間都會有一條 connection，而每一條 connection 又會產生兩個 thread，分別負責send/write的行為。
所以當 connection 數量一多的時候，會產生非常多的 thread，然後每個 thread 都各自在自己的迴圈內去處理自己的事情。這種網路程式設計的方法到現今幾乎都被 **event-based** 的方法給取代，當有事件發生時再來處理，而不是開一個 **thread** 在那邊癡癡的等待封包處理。
若要使用這種架構，則將 **ms_type** 修改為 **simple** 即可。


#### Xio
Xio比較不算是原生的功能，其用到了第三方的library [Accelio XIO](https://github.com/accelio/accelio)。
**Accelio XIO**是一套提供穩定，高速網路資料交換的函式庫，除了支援常見的 **Ethernet Network** 外，也有支援 **RDMA**, 所以現在網路上看到很多 **ceph** 關於RDMA的效能測量都是基於使用 **Xio**作為其網路傳輸的實現。
然而 **ceph** 後來也沒有在維護使用**xio**作為其網路傳輸的一部份，主要還是因為要引入第三方函式庫，不是直接使用 **ceph**本身的架構，同時若有任何bug出現，除非等待**xio**修復，否則開發人員還要再花時間理解一套第三方函式庫，並且嘗試解決問題，這在整理維護成本上是不容忽視的。
若要使用這種架構，則將 **ms_type** 修改為 **xio** 即可。

#### Async
相對於 **simple** 採用兩條 **thread** 專門負責收送動作的做法，**Async**則是採用 **event-driven**的方式來處理請求，整體設計上會有一個**thread pool**，會有一個固定數量（還是可透過API動態調整)的 **thread**。
然後將每一條 connection 分配到一個 thread 身上，然後透過監聽該**connection**的狀況，當底層觸發了**收**或是**送**的事件時，更精準的應該是講 **read**/**write**，就會呼叫對應的函式來處理相關的行為。
此種設計可大大減少系統上開啟的 **thread** 數量，減少系統的消耗。
在此設計下， **ceph** 又提供了三種模式，分別是 **POSIX**, **DPDK** 以及 **RDMA**。
這三者的切換方式分別是將**ms_type**修改為
- async
- async+dpdk
- async+rmda

##### POSIX
POSIX代表的就是走 **kernel** 大家熟悉的**event**機制，譬如**select**, **epoll** 以及 **kqueue**.
同時此類型也是 **async**的預設類型，因為此類型完全不需要任何硬體的幫忙，完全是靠**software**的方式就可以完成的，主要是看 **kernel** 本身的支援程度來決定實際上會呼叫出哪一種實現方式來使用。
##### DPDK
Data Plane Development Kit(DPDK)是一套 **intel**所主導的技術，基本上就是使用**CPU**換取**效能**的機制，藉由此機制，**user-space**的程式可以略過 **kernel** 直接跟硬體溝通，這部分採用的是**polling**的方式，藉由不停地詢問來加強整理的效能，然而也會因為一直不停的**polling**使得**cpu**使用率提升。
然而此技術只要是**x86**的 CPU 基本上都支援，所以在使用上其實可以說是非常的廣泛，很容易被支援，不太會有被特定硬體綁住限制的機會。

##### RDMA
RDMA代表的是遠方記憶體存取，是一種擁有**low latency**, **low cpu consumption**, **kernel by pass**等特性的傳輸方法，藉由跳過**kernel-space**的方式，讓整體資料流量直接透過網卡去處理，同時也可以直接針對遠方的記憶體進行修改而不會讓遠方的**CPU**察覺這件事情。
一般來說支援的網路底層有 **Infiniband** 以及 **Ethernet**，這部分由於封包會忽略 **kernel space**，因此資料在 **internet** 上傳遞勢必要符合當前廣泛的格式，譬如 **Ethernet**，因此這邊會採用 **ROCE** 的方式來處理封包之間的標頭問題。
目前 `ceph` 上面已經可以運行 **RDMA**，不過根據開發者在 **Mail** 中的說法，目前還是在尋求穩定性為主，效能上還沒有達到最佳化的部分，因此使用起來與 **POSIX w/ TCP/IP** 在各方面都不會有太明顯的提升。

在看完這些網路底層的實現後，接下來要來探討**Network**是如何提供介面給其他元件，如**OSD**,**Mon**等使用的。

### Architecture
整個網路最基礎的架構程式碼都放在 **/src/msg/** 裡面，排除三種類型的資料夾外，大致上就是下列四種類型
- Message 
- Connection
- Messenger
- Dispatcher

而剛剛上述提到的那些種類，其實就是繼承這些基本架構，並且實現了每個介面的功能而已。
因此接下來會比較偏向概念性的去分析這四種概念的用途。

#### Messege
此物件主要用來定義封包的格式，所有 **ceph node**之間的封包傳送都必須要參照此格式，不過對於應用層(osd,mon等)不需要擔心這邊，這邊是由網路層去負責包裝跟解析的

#### Connection
代表任意兩個 **ceph node** 之間的連線，彼此之間可以傳送/接收封包。
#### Messenger
此物件用來管理連線，一種Messenger可以管理多條連線，目前在 osd 的使用中，是採用**一種**類型的連線使用一個 **Messenger**，而底下可以有很多條**connection**。
舉例來說，今天有兩種連線類型分別代表 **heartbeat public** 以及 **heartbeat cluster**，且環境中有 三台**osd**，所以於 **mesh** 的架構中，總共會有六條連線，(每台 osd 彼此互連，且都有兩種類型的連線)。
在**heartbeat public**的 **messenger** 會採用 **public network**去建立 connection，而在 **heartbeat cluster** 的 **messenger** 則是會採用 **cluster network** 去建立 connection。
這邊的範例剛好是使用不同網路類型的 *messenger**，實際上也可以是不同用途的，譬如用來傳遞 **heartbeat**，用來傳送 **control message** 或是用來傳送 **data message**之類的。
這邊的架構如附圖

![](http://i.imgur.com/viZA6UG.jpg)


可參考 **OSD** 實際創造這些 **messenger** 的程式碼
```c++
  Messenger *ms_public = Messenger::create(g_ceph_context, public_msgr_type,
					   entity_name_t::OSD(whoami), "client",
					   getpid(),
					   Messenger::HAS_HEAVY_TRAFFIC |
					   Messenger::HAS_MANY_CONNECTIONS);
  Messenger *ms_cluster = Messenger::create(g_ceph_context, cluster_msgr_type,
					    entity_name_t::OSD(whoami), "cluster",
					    getpid(),
					    Messenger::HAS_HEAVY_TRAFFIC |
					    Messenger::HAS_MANY_CONNECTIONS);
  Messenger *ms_hb_back_client = Messenger::create(g_ceph_context, cluster_msgr_type,
					     entity_name_t::OSD(whoami), "hb_back_client",
					     getpid(), Messenger::HEARTBEAT);
  Messenger *ms_hb_front_client = Messenger::create(g_ceph_context, public_msgr_type,
					     entity_name_t::OSD(whoami), "hb_front_client",
					     getpid(), Messenger::HEARTBEAT);
  Messenger *ms_hb_back_server = Messenger::create(g_ceph_context, cluster_msgr_type,
						   entity_name_t::OSD(whoami), "hb_back_server",
						   getpid(), Messenger::HEARTBEAT);
  Messenger *ms_hb_front_server = Messenger::create(g_ceph_context, public_msgr_type,
						    entity_name_t::OSD(whoami), "hb_front_server",
						    getpid(), Messenger::HEARTBEAT);
  Messenger *ms_objecter = Messenger::create(g_ceph_context, public_msgr_type,
					     entity_name_t::OSD(whoami), "ms_objecter",
					     getpid(), 0);
```


#### Dispatcher
Dispatcher 這邊的概念簡單來說就是**當接收到封包後，要怎麼處理**，每個應用層(OSD/MON.等)創建 **Messenger** 後，要向其註冊 **dispatcher**，這行為解讀成，當該 **messenger** 內的 **connection** 有從對方收到訊息後，所要執行的對應 **function**。
該 **function** 原型內可以判別是由哪一條 **connection**所觸發的。
 ![](http://i.imgur.com/fpXuDmD.jpg)
a若要註冊該 **dispatcher**，可參考 **OSD** 實際程式碼如下

```c++
  client_messenger->add_dispatcher_head(this);
  cluster_messenger->add_dispatcher_head(this);

  hb_front_client_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_back_client_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_front_server_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_back_server_messenger->add_dispatcher_head(&heartbeat_dispatcher);
```



最後做一個簡單總結，每個應用層可以自由創建 **messenger**，並且向其註冊對應的 **dispatcher**，同時使用該 **messenger**去管理該類型的多條連線。

![](http://i.imgur.com/cvot2Ow.jpg[)
至於底層是如何連線，封包如何收送，訊息如何解析、封裝，都些都是 Networking 本身的事情，就是本文開頭提到的那幾種方式去實作。

因此接下來的文章，將探討 **Async** 這類型的傳輸方式是如何將上述的概念給實作的。

