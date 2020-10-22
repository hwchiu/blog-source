---
title: '[論文導讀] Maglev: A Fast and Reliable Software Network Load Balancer'
keywords: 'Maglev, google, lb, load balancer'
tags:
  - Paper
  - Network
  - Linux
  - Kernel
abbrlink: 59612
date: 2017-08-29 11:36:49
description: 本篇文章是屬於論文導讀系列，這次的對象是Google所推出的Software Network Load Balancer, Meglev. 透過對該論文的研究後可以學習到Google對於一個 Network Load Balancer 的期許以及設計的思考脈絡，並且實際理解其架構來學習到如何設計一個通用(可運行在任意的 Linux Server上), 分散式且易於擴充的彈性架構以及高PPS(Packet Per Second)處理能力的軟體程式。最後透過論文中的實驗與效能評估來觀察實際上 Meglev 的效能以及是否有滿足Google對該軟體架構的期望。

---

# Preface
最近才看到一篇由**Google**於2016年所發佈的論文[Maglev: A Fast and Reliable Software Network Load Balancer](https://static.googleusercontent.com/media/research.google.com/zh-TW//pubs/archive/44824.pdf)，其標題看起來也頗有趣的，加上最近有些時間，就決定花點時間看一下這篇論文，順便用這篇文章記錄一下論文中的重點，當作一個紀錄。


# Abstract
Google 自從 2008 年開始就在其內部的網路中設計了一套名為 **Maglev** 的網路負載平衡器。Google 對於 **Maglev** 有者下列的期許
1. 分散式的軟體架構，能夠輕易地增加/移除其機器並且彈性的調整其服務能力
2. 能夠運行在任意的 Linux Server上
3. 每秒鐘處理的小封包數量盡可能地高 (pps)

因此接下來將會講述 **Maglev** 的架構，並且敘述一下其實作的原理，最後附上該論文內的一些實驗數據來檢視其成果。


# Introduction
由於Google本身於全世界提供了大量的服務，這些服務無時無刻都有大量的請求在網路上流動，為了讓這些流量能夠在這些服務群集中有一個很好的分配，因此需要再網路架構中安插所謂的負載平衡器，而下圖則用來說明傳統上認知的負載平衡器與 **Maglev** 的不同。
![](http://i.imgur.com/bby13s6.png)

# Traditional LoadBalancer
傳統的負載平衡器幾乎都是由一台特定的硬體服務器來處理，著名的廠商譬如 F5。
而這類型的負載平衡器為了達到高可用性(HA)，所以在建置上通常都會部署兩台機器，採取AP(Active-Passive)模式來運行。這樣的優勢就是當一台機器壞時，另外一台能夠接取其任務繼續服務。
然而其缺點就是其能夠服務的能力不夠大，整個處理速率都受限於一台機器本身，同時不論後端的服務機器數量有多少，前面都只有兩台負載平衡器再處理，因此整體服務的效率低落。
最後，其本身沒有彈性且沒有辦法透過程式去進行修改，在整個更新與變化的需求上沒有辦法滿足Google如此快速成長的環境。


# Maglev
**Maglev** 期待中的角色就如同上圖右半邊所示，**Maglev**希望是個分散式的軟體架構，可以根據背後的需求來動態的調整**Maglev**的數量，同時所有的機器都能夠同時用來處理所有的流量需求。
在這種架構下，Google 就可以針對本身服務的大小來調整 **Maglev** 的數量，同時可以依據各種不同的情境來客製化。

## System Overview

### Frontend Serving Architecture
接下來使用下圖來說明一下整體的系統架構
![](http://i.imgur.com/Bb8Byvy.png)

Google本身的服務，譬如 Gmail, Google Search本身都含有一組或是多組以上的 **IP address**,而這些 **IP address** 被稱為 **Virtual IP**(VIP)。原因是這些 **IP** 本身是不存在於任何實體網卡上，只是讓網際網路中的路由器能夠根據這些 **VIP** 將這些封包給導向到 **Google** 的服務器之中。接下來這些封包就會傳送到 **Maglev**這群服務器中去處理，再根據VIP找到對應的服務，然後把封包傳送給真正的服務器去處理。

假設以 **Gmail** 為範例，當使用者要連結到 **gmail.com**的時候，會先到 **DNS** 本身去詢問 **gmail.com**所對應的 **IP address**。
而這些 DNS 回應的 **IP** 對於 **google** 來說其實是 **VIP**，然後使用者的電腦都會嘗試將請求的封包送到 **VIP** 所對應的路由器去處理。
當 **Router** 收到這個 **VIP** 的封包後，接下來他要把這個封包送到底下的 **Maglev** 服務器群去處理，在 **Router** -> **Maglev** 的過程中採用了 **Equal Cost Multi Path**(**ECMP**)的方式去傳送封包，盡可能的讓這些 **Maglev**服務器能夠平均的收到請求封包，這邊在我看來也是一種簡單的負載平衡的功能，不過著重的對象是**Maglev**而不是背後真正服務的服務器。

當 **Maglev** 收到這封包的時候，他就會根據目的地的 **VIP** 去反查，就可以知道當前這個封包應該要往哪個 **service**傳送，但是我們知道因為 **VIP** 本身是不存在的，所以這時候 **Maglev** 會幫當前整個封包再多包上一層 **Generic Routing Encapsulation(GRE)** 的標頭檔，該標頭內的資訊則是後端服務器真正的 **IP address**，因此封包就能夠順利的到達後端服務器，這也是圖中所標示**Encapped inbound traffic**的流向。

當後端服務器處理完畢請求時，這時候會回傳封包到發送端(也就是使用者電腦)，這邊有個要注意的事情是通常情況下, **使用端發出的請求封包**會比**服務器發出的回應封包**還要來得小很多，因此 **Maglev** 並不想要讓這些回應封包還要回歸到 **Maglev** 去處理。
所以當服務器收到封包後，要先解讀GRE，接下來讀取到本來的 **VIP**,然後將此 **VIP** 當作封包的來源IP後讓該封包直接送回給使用者端。
所以就如同該圖示中三號紅色的 **Unencapped outbound traffic**。
至於要如何讓這些封包能夠從服務器本身不經過 **Maglev** 而直接到達上層的 **Router**，這邊論文內本身並沒有說明，只有提到透過 **Direct Server Return** (**DSR**)的技術來達到此功能。

最後要提到的是這些 **VIP** 為什麼可以被網際網路中的路由器給導向過來，原因是上圖中的咖啡色 **BGP announcements**，當後端服務跟對應的 **VIP** 有任何更動時，都會通知到 **Maglev**然後透過 **BGP** 的方式一路傳遞到網際網路去更新動態路由表，讓流向**VIP**的封包都能夠順利的導向內部服務器。

## Maglev Configuration
接下來使用下圖來說明整個 **Maglev**內部的設定。
![](http://i.imgur.com/RGbeovC.png)
如同前面所述， **Maglev** 本身會透過 **BGP** 的方式向路由器去通知路由表的更新，因此在 **Maglev** 內部會有兩個元件，分別是 **Forwarder** 以及 **Controller**。
這兩個元件會透過 **Config Objects**來學習 **VIP**的資訊，可以是透過讀取檔案的方式，或是透過 RPC 的方式來更新。

**Controller** 會定期檢查 **Forwarder** 的資訊，只要當前有任何 **VIP** 變動，不論是新增或是減少，都會透過透過 **BGP** 將 **VIP**的更動一路往外宣傳，根據這種舉動可以確保 **Router** 要轉發封包時，一定會轉發到能夠正常運作的 **Maglev** 伺服器。

而當 **VIP** 的封包到達 **Maglev** 時，都會交由 **Forwarder** 來處理，對於 **Forwarder** 來說，每個 **VIP** 可以對應到一個或多個的 **Backend Pool**，而每個 **Backend Pool** 可能包含一組或多組的實體 **IP**，而這些**IP**則會對應到後方真正的服務器 **IP**。
對於每個 **Backend Pool**來說，背後都會對應到一組或多組的 **Health Checker**，這些 **Health Checker**會去檢查該 **Backend Pool**內所有的服務是否當前都運作正常，只有都運作的正常的**Backend Pool**才會被 **Forwarder** 視為一個封包轉送目的地的候選人。
根據此架構圖，其實可以看到有些後方服務器(**IP**)是對應到多組的 **Backend Pool**，所以在 **Health Checker**的時候會特別去進行這邊的去重複化，避免相同的事情重複多次來減少額外的開銷。


# Forwarder Implementation
由於 **Forwarder** 要負責接收封包並轉發，所以必須要有極高的效能且穩定，所以接下來就會介紹 **Maglev** 內部的架構以及其實作原理。

## Overrall Structure
接下來會使用下圖來介紹整個 **Forwarder** 處理封包的過程。
![](http://i.imgur.com/86K4rvQ.png)
整個過程簡單來說就是
- 從網卡(NIC)收到封包
- 經過查詢計算後，透過 **GRE** 重新封裝該封包
- 從網卡(NIC)送出封包

複雜來看的話，整個 **Forwarder** 分為兩個大模組來處理，分別是 **Steering** 以及 **Muxing**。
首先, **Steering** 模組會針對封包的 **5-tuples** 進行 **Hash**，根據 **Hash** 的值找到一個對應的 **RX Queue**來處理，而每一個 **RX Queue** 都會對應到一個 **Packet Rewriter Thread** 來處理。
而 **Packer Rewriter Thread** 則是會進行下列事情
1. 先排除根本機無關的封包，譬如 **VIP** 不屬於本機上面的封包
2. 重新根據 **5-tuples** 進行一次 **Hash** 計算，然後根據此計算其果去查詢一個 **Connection Track Table** 表格。**Connection Track Table** 會記住每條 **Connection** 與後端服務器的對應關係。
這邊要重新計算 **Hash**的原因是因為不想要跟 **Steering** 共用一樣的數值，因為這樣就會有跨**Thread**之間的同步問題，這樣就會導致效率降低。
3. 當查詢 **Connection Track Table** 時
    - 若 Hash 存在且對應的後端服務器依然正常服務，那就直接使用查詢出來的結果當作當前 **VIP**封包要轉送的對象
    - 若 Hash 不存在或是後端服務器目前服務有問題，則會透過 **consistent hash**的方式算出對應的後端服務器位置，並且將其加入到該表格之中。
**Consistent Hash** 後面章節會再介紹。
    - 若目前查詢的結果是沒有半個後端服務器可以使用，則就丟棄當前封包。
4. 當知道當前 **VIP** 所要對應的服務器資訊後，就會透過 **GRE/IP** 的方式重新改寫當前封包內容，並且將封包送到 **TX Queue**

接下來 **Muxing Module** 則會定期 **Polling** 所有的 **TX Queue**，然後將封包從 **TX Queue**中取出來並且透過 **NIC** 將封包給轉送出去。

最後有提到為什麼 **Steering Module** 這邊要特別採用 **Hash** 的方式來選擇 **RX Queue** 而不是透過 **Round-Robin** 的方式。
有兩個原因
1. 讓相同 **Connection** 的封包能夠由同一個 **Queue**去處理，避免因為不同 **Queue** 之間處理速率不同，導致相同 **Connection** 的封包以不同順序的結果從NIC出去，這對於 **TCP** 來說有可能會導致 **Out of Order** 的現象進而導致降速。
2. 對於 **Connection Track Module** 來說，每個 **Connection** 只需要在一個 **Packet Rewrite Thread** 去計算就好，這樣可以避免同樣結果多次計算藉此降低 CPU 使用率，同時也可以避免同一條 **Connection** 最後算出不同後端服務器導致該 **Connection** 出問題。

不過最後也有提到，若當前選擇的 **RX Queue** 沒有空間的情況下， **Steering Module** 則會自動切換成 **Round-Robin** 的模式來處理封包。

## Fast Packet Processing
在**google**內部普遍存在使用於 10Gbps 的網卡來說，假設每個 IP 封包的大小是 1500 byte, 則最高的情況下每秒處理封包數量就是 813 Kpps。但是由於使用者發送來的請求封包都會更小，所以假設每個請求的 IP 封包只有 100 byte, 則每秒收到的封包數量可高達 9.06M pps,這個數量非常可觀。
同時，**Maglev** 是個運行在一般 Linux Server 上的 **User-space** 應用程式，其實本身的功能並沒有需要 **Linux Kernel Network** 內這麼龐大的功能，所以這邊促使了 **Google** 提出了採用 **Kernel Bypass** 的架構來設計 **Maglev**，透過適當的技術(譬如 **DPDK**, **Netmap**)等技術，讓網卡(NIC)收到的封包不再需要經過 **Linux Kernel Network Stack**, 對於每次的收送來說，都可以減少至少兩次以上的 **kernel-space** 與 **user-space** 的資料複製操作。
在 **kernel bypass** 的架構下, 整個 **Maglev** 內 **Forwarder** 的架構就如下圖所示。
![](http://i.imgur.com/JoKiFAk.png)
當 **Maglev** 一旦啟動後，會預先產生一塊很大的空間 **Packer Pool** 供 **Forwarer** 與 網路卡(NIC) 共同使用。
**Steering** 與 **Muxing** 都各自維護一個 **Ring Queue**，其內容則是各種指標，這些指標會直接指向 **Packer Pool**內的每一塊空間。

在 **Ring Queue** 內則維護了三個指標，分別是 **Processed**, **Received** 以及 **Reserved**。
對於接收端來說，當網卡收到封包後，就會將 **Received** 指標給移動，然後記住哪些封包目前已經收到。
接者 **Steering** 就會開始處理這些封包，只要 **Sterring Module**一旦將該封包送給 **Packet Rewrite Thread** 去處理後，該封包就會被標示為 **Processed**，並且繼續往下移動指標，直到遇到 **Received**指標。
**Reserved**則是會從 **Packer Pool** 中預先拿一些還沒被使用的空間出來，供之後的 **Received** 使用。
相對於接收端，發送端其實概念也很類似。
當網卡把封包發送出去後，就會修改 **Sent** 的指標，而 **Ready** 則是 **Muxing Module** 將封包從 **TX Queue**取出後就會被更動。

這邊 **google**有特別強調的是在這些 **Forwarder** 的操作中，沒有任何一個封包是會被複製的，這意味者每個封包能夠減少大量的複製操作，藉此減少每個封包處理所需要的時間。

此外，在軟體架構方面，讓每個 **Packet Rewrite Thread** 擁有下列特性
1. 彼此沒有任何資料要互相同步，避免同步產生的時間消耗
2. 每個 Thread 都運行在不同的 CPU 上

最後，想來探討在這種架構下，每個封包處理所花費的時間。
一般來說，每個封包大致上花費 350ns 來處理，不過有兩種特殊的情況可能會導致該時間變大。
首先要先瞭解到的是 **Maglev** 處理封包的方式是採用 **Batch** 的方式去處理，每次都會批量的處理直到一個固定數量或是該處理花費的時間已經達到了定時器的數值。

第一種情況
假設當前 **Maglev** 服務器處於一種沒啥流量的情況下，同時上述定時器的數值假設是 **50us**, 則最壞的情況下，每個封包的處理都至少要花到 50us 才可以處理完畢。
這種情況的解決方法可以是動態的去調整該該定時器的數值 (**50us**)

第二種情況
假設當前 **Maglev** 服務器處於一種高負載的情況下，當前存放封包的 **Ring Queue** 已經滿了，這時候多出來的封包都會被丟棄。
假設 **Ring Queue** 能夠存放 **3000** 個封包，同時 **Maglev** 處理封包的能力是 10Mpps, 則處理完這 **3000** 個封包則要額外花費 **300us**，所以每個封包處理所花費的時間不但是本身處理的時間，還要再加上在**queue**中等待的時間。
這種情況的解決方法可以透過增加更多的 **Maglev** 來平衡每台服務器所承受的壓力。

## Backend Selection

當 **Maglev** 收到一個 **VIP** 的封包時，要如何決定該 **VIP** 封包對應到後端的服務器是誰。對於 **TCP** 這種有連線概念的協定來說，若相同連線的封包沒有到達同一個收端，會對效能產生不好的影響，譬如 **Out of Order**。 因此要如何讓每條 **Connection** 內的所有封包都能夠選到同一個後端的服務器，就是這章節強調的重點。

就如同先前所述，每一個 **Maglev** 內的每個 **Packet Rewrite Thread** 都會自己維護一個對應的 **Connection Track Table**。
對該 **VIP** 封包根據 **5-tuple**進行 **Hash** 運算後，
若該 **Hash**值已經存在，則使用先前紀錄過的服務器位置來使用，否則就會根據 **Consistent Hashing** 的方式算出對應的後端並且將此結果存到對應的 **Connection Track Table** 中。

然後上述的這種架構，對於分散式的 **Maglev** 是不夠的，這邊舉了兩個範例。
第一個範例
因為 **Maglev** 是能夠動態的加入/移除/升級的一種分散式架構，所以搭配前述的 **ECMP** 路由演算法，同樣一條 **Connection** 可能會被導向不同的 **Maglev** 服務器，在此情況下，因為該台機器本身對於該 **Connection** 沒有任何記錄，所以必須要重新計算其對應的後端服務器是哪些。

第二個範例
因為 **Connection Track Table** 本身的大小有限制，所以假如該 **Table** 因為遇到大量的流量或是 SYN Flood 攻擊之類的導致 **Table** 滿載，這時候新加入的 **Connection** 就沒有辦法記錄下來，導致每個封包都要重新計算一次。

**Google**特別設計了一套 **Consistent Hashing** 來處理這上述這兩種情況，讓上述情況內的封包還是依然可以算出相同的後端伺服器以避免 **Connection** 出問題。

## Consistent Hashing
這邊來說就是在說明假設當 **Maglev** 有任何機器壞掉的話，我要如何確保其他的 **Maglev** 服務器在進行 **Connection Track Table**建立時，能夠將相同的 **Connection** 給算到相同的後端服務器。
因為不想要服務器彼此之間有任何同步的行為，這些都會額外的支出都會減少每秒封包處理的速度。

這部分網路上已經有一篇文章在解釋這邊的行為，並且解釋得淺顯易懂，建議能的話一定要看一下這篇由 EvanLin 所撰寫的
[[論文中文導讀] Maglev : A Fast and Reliable Software Network Load Balancer (using Consistent Hashing)](http://www.evanlin.com/maglev/)


# Operational Experience
這邊大致上就是一個更細節的探討，包含 **Meaglve** 的演化史，**VIP** 怎麼設計，遇到 **IP Fragement**如何處理 以及一些 **Monitor**的設計。
這邊有興趣的可以自行閱讀該篇文章，這邊就不多加敘述。


# Evaluation
這邊效能評估的部分，我個人偏好 **Kernel Bypass**的部分，所以這邊只針對這邊去進行閱讀。

在此實驗中，變數總共有兩個，分別是
1. Linux Kernel Stack/Kernel Bypass
2. **Packet Rewrite Thread** 的數量

希望觀察到的是 **Maglve** 每秒能夠處理的封包數量。

實驗環境中，發送端(Sender)會產生不同 **Source Port** 的 UDP 封包，讓這些封包都會被分配到不同的 **Packet Rewrite Thread**。
每個 UDP 封包的大小都是 52 Byte，然後 **Maglev** 上的 每個 **Thread** 都會綁上一個專屬的 CPU 來處理。

實驗結果如下圖，基本上總結是
1. **Thread** 數量超過四以後就不會再上去了，這時候整個效能的瓶頸就從 **Meaglve** 轉移到 **NIC** 上
2. Kernel Bypass 相對於 **Linux Kernel Network Stack** 有明顯大服務的差距
![](http://i.imgur.com/K7aeCZH.png)


# Reference
[[論文中文導讀] Maglev : A Fast and Reliable Software Network Load Balancer (using Consistent Hashing)](http://www.evanlin.com/maglev/)

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