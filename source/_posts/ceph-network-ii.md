---
title: Ceph Network Architecture 研究(二)
date: 2017-05-25 10:15:21
tags:
	- Ceph
	- Network
	- SOS
	- ScaleOut
	- SDS
	- SourceCode
keywords: Ceph,Network,SourceCode,SDS,SOS
---

延續上篇文章 (Ceph Network Architecture 研究(一))[https://blog.hwchiu.com/ceph-network-i.html#more]，本文將繼續探討 **Async** 這種網路類型底層真的架構與概念，所以本文章也不會有太硬的程式碼解讀，反而會比較偏向概念性的分析。

從上一篇文章中我們可知道，底層網路實現提供了包含 **Messenger**, **Connection** 等介面供上層應用層去使用。
接下來

```
AsyncConnection.cc
AsyncConnection.h
AsyncMessenger.cc
AsyncMessenger.h
Event.cc
Event.h
EventEpoll.cc
EventEpoll.h
EventKqueue.h
EventSelect.cc
EventSelect.h
PosixStack.cc
PosixStack.h
Stack.cc	
Stack.h	
net_handler.cc
net_handler.h
```

從上述的檔案來看，我們大致上可以猜到 **AsyncMessenger** 以及 **AsyncConnection**的涵義，而 **Event** 一系列的檔案應該就是最底層的 I/O 的處理。
<!--more-->

## Event
這邊先從最底層開始看起，首先是 **Event**。
整個 **Event** 代表了底層 I/O 的處理，目前支援三種實作方式，分別是
- Epool
- Kqueue
- Select

在這三種實作上面，又提供了三種 **EventType** 供上層使用，分別是
- File_Event
- Time_Event
- External_Event

### File_Event
File event 是最常見也是最普遍使用的，針對每一個 **file descriptor** 進行設定，該 **file descriptor** 關注的是 **read event** 還是 **write event** 以及當該 **event** 發生時，應該要進行什麼樣的處理。

### Time_Event
Time event 本身跟 **file descriptor** 無關，單純的是依照時間來驅動的事件，對於每一種 **Time Event** 則會有兩個設定，一個是多久(ms)之後要執行該事件，以及該執行的事件是什麼，這邊都是使用 function pointer 來指向該事件。

### External_Event
External event 實際上跟 Time event 是相同的，可以視為時間是 0ms 的 Time event，就是馬上執行該事件。

整體架構如下圖，系統中會有一個 **Event Center**，底層支援各種 I/O 的實作，此外，本身會提供介面供上層使用，可以接受上述三種事件的註冊。
待一切都準備就緒後，就會開始透過底層 I/O 的事件去處理這三種事件，譬
如有封包到來的時候呼叫對應的函式處理，亦或是時間到的時候執行對應的
 **Time event**。

![](http://i.imgur.com/yiDGubn.jpg)

### Worker and Network Stack
上述看完了 **EventCenter**的概念後，我們知道每個 **EventCenter**專門用來負責底層的 I/O 處理，那這邊為了提高整體的效率，採用了 **ThreadPool**的概念，事先根據系統上的能力創造一批固定數量的 **Thread**，這邊統稱為 **Worker**，這些 **Worker** 的數量可以動態的增加，減少。
然後每一個 **Worker** 都配上一個 **EventCenter**，盡可能的讓所有的 **Worker**去平均分擔所有的 Network I/O 負擔。
在這群 **Worker** 之上存在一層 **Network Stack**，此 **Stack** 會掌管所有的 **Workers**，包含其創建/增加/刪除等行為。

綜合以上概念，目前認知的架構圖如下。
![](http://i.imgur.com/4k6Rmpo.jpg)

### AsyncConnection
接下來看到 **AsyncConnection** ，此物件代表者任意兩個 **ceph node**之間的連線，可以是 **osd<->osd**，也可以是 **mon<->mon**亦或是 **osd<->mon**。
此物件本身除了代表連線外，跟網路傳輸相關的功能，如發送封包，接收封包等事情都會在這邊提供一個介面供更上層的應用(OSD)來使用。
不過這邊只是中介層而已，真正收送發包的還是上述提到 **EventCenter**
在處理。

由於系統上可以同時存在非常多條 **connections**，為了讓這些 **connections**能夠同時運作且不會互相影響，這邊配給**每一個connection**一個**worker**來處理。
由於 **connection**的數量基本上都會比**worker**還要多，因此在配對上就是盡量平均分擔下去，盡量讓每一個**worker**負擔相同數量的**connection**。

所以看到這邊已經可以大概理解，系統上每一條 **connection** 都配上一個 **thread** 來處理，而每一個 **thread** 實際上可能會負責不少條 **connection**，這邊採用數量的方式來分散這些 **connection**，若能夠根據實際負擔作為分散的權重也許可以讓每個
 **thread** 的負擔更為平均。

因此看到這邊，目前的架構圖如下，圖中的 **Async Connections**只是一個抽象的概念，描述眾多的 **Async Connection**而已。

![](http://i.imgur.com/yAzGB06.jpg)


### AsyncMessenger
最後出場的就是 **AsyncMessenger**，其概念與之前講述過的 **Messenger**一樣，管理所有的 **AsyncConnections**，同時是最直接供上層應用程式的物件。
每個 **AsyncMessenger**都可以被註冊 **Dispatcher**，接者當底下的 **connection** 有收到來自遠方的封包(**Messege**)時，就呼叫對應的 **Dispatcher** 來處理。

所以整體架構如下圖。

![](http://i.imgur.com/l9h13kc.jpg)

## Summary
到這邊大致上已經對 **Async** 系列有一個基本的認識了，不過這基本上只局限於 **async+poxis** 類型而已，對於 **RDMA** 以及 **DPDK** 則因為這兩種運作方式的不同，其真正運作的方式又是截然不同的，這部分可以從原始碼中看到 **RDMA/DPDK** 都有額外的資料夾，且資料夾內又有為數不少的檔案可以大略猜測出來。

接下來的文章會比較偏技術性質，會直接看進這些程式碼內，透過這些程式碼能夠更瞭解整體的架構以及實作細節，順便學習增廣見聞。
