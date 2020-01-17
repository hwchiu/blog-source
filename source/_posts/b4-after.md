---
title: "[閱讀筆記] B4 and After: Managing Hierarchy, partitioning, and Asymmetry for Availability and Scale in Google's Software-Defined WAN"
keywords: 'Google, B4, 2018, Sigcomm'
abbrlink: 45750
date: 2019-01-19 14:47:57
tags:
  - SDN
  - Network
description: 本篇文章主要的概念是閱讀筆記, 主要是針對 Google 於 2018 Sigcomm 所發表關於 SD-WAN 的相關論文,這篇論文非常直得一看的點是這篇論文算是 2013 Sigcomm B4 論文後的後續，講述了 SDN 概念引進 B4 帶來的好處以及這幾年因應環境變化而該 B4 資料中心的成長，其中包含了眾多的問題以及處理的方式，著實非常有趣，能夠學習到更多的想法與概念

---

# Preface
有在長期關注 SDN 的朋友們，可能有聽過 `B4` 這個名詞，甚至相關的論文 `B4: Experience with a Globally-Deployed
Software Defined WAN`. 這是一篇 Google 於 `2013 SIGCOMM` 所發表關於 `SDN` 應用的一篇論文. 就`我個人`觀點來看，這是一篇劃時代且極具意義的論文，在那個 `SDN` 受到大量學界研究但是業界不看好的年代，由一個網路巨人 `Google` 站出來發表論文闡述將 `SDN` 的概念引進到 `B4` 這個跨國資料中心叢集中所帶來的好處及優點，無疑是對整個 `SDN` 的發展打入一劑強心針。

而今天這篇論文，則是 `Google` 睽違五年後於 `2018 SIGCOMM` 所發表的論文，探討 `B4` 這數年來的變化，包含了遇到的問題，架設的設計以及解決的方案。 對於一個 `SDWAN` 的應用來說，其中關於 `TE(Traffic Engineering)` 的想法非常的有趣且難懂，非常值得好好仔細咀嚼來學習巨人的經驗。

本文主要是針對下列兩份文件進行閱讀後所撰寫的簡易筆記，有任何錯誤或是不慎瞭解的地方請盡量，並且歡迎指正
1. [B4 and after: managing hierarchy, partitioning, and asymmetry for availability and scale in google's software-defined WAN](https://dl.acm.org/citation.cfm?id=3230545&fbclid=IwAR3JnZf8yk2Ku8JXutQZSsAwm8Koca2ZSlInGI7JOJb9P3rAu91EWHai2c0)
2. [Slides](https://pdfs.semanticscholar.org/63f9/0f7482c186b778ef32b70c877c6d7ec82440.pdf?fbclid=IwAR06jKF1W8bXwga9aRzeVYMEMMXn63vzc6dm1V7OSrJfyquoAUbwNgYmKEE)

{% note info %}
強烈建議一邊閱讀的同時也閱讀該份投影片，會比較容易比較
{% endnote %}


# What is B4
由於本文的重點在於 `B4` 這數年來的發展與介紹，所以這邊會粗略的大概介紹一下到底 `B4` 是什麼，以及第一篇 `2013` 的論文中到底闡述了什麼樣的概念。 有了這些基本的概念，對於後續的發展才能夠有更深的體悟與想法。

- B4 是跨國的資料中心叢集，在 2012 年時全世界總共有 `12` 點 (Private WAN)
    1. 到了 2018, 節點數量已經擴增為 33 個節點
- 透過 OpenFlow 來控制叢集中的交換機，同時透過 `Controller` 的計算來達到 TE(Traffic Engineering) 的效果
- 透過 `TE` 的幫忙，提升整體網路頻寬的使用率到達接近 100%. 同時可以根據應用程式的優先性與需求來排定其連線，針對高優先度的服務可以提供盡可能低的延遲，反之亦然
- 透過 `TE` 達成的高頻寬使用率，一方面也代表降低整體 `per-byte` 的建置費用，在整體的建置與使用上相對於其他的資料中心(B2)來得更有效率
- 透過集中化的 `TE` 計算與控制，能夠很快速的去處理任何錯誤
- 快速的軟體迭代，一個月左右的時間就能夠開發並且部署新的軟體功能

# Challenge
隨者這些年 Google 業務的發展， `B4` 遇到了不少的挑戰與問題
1. High Availability 的需求提升. `2013` 年時 `HA` 需要達到 99%, 到了 `2018` 年，整個服務的種類提昇，而 `HA` 的要求從 `99%` 一路到 `99.99%`

目前總共分成 `五` 類，這五類有代表性不同的服務並且都有各自的 `HA` 要求。 而下圖的 `SLO HA` 的定義是該服務在 `30`天內必須要有 X% 的分鐘數是可以使用的。
所以 `99.99%` 就意味大概每三十天只有`4.32` 分鐘不能使用。
至於`可以使用`的定義則是
1. network connectivity (封包遺失率必須要在某個標準以下，本文沒有提及該標準)
2. promised bandwidth



| Service Class  | Application Examples Availability | Avail. SLO(Service Level Objectives) |
| -------- | -------- | -------- |
| SC4     | Search ads, DNS, WWW     | 99.99%     |
| SC3     | Photo service backend, Email   | 99.95%     |
| SC2     | Ads database replication    | 99.9%     |
| SC1     | Search index copies, logs    | 99%     |
| SC0     | Bulk transfer   | N/A     |

2. Scale Requirement
根據 `Google` 自己提供的數據，整體網路所需的頻寬每九個月就會成長一倍，根本是另類的摩爾定律(X). 因此這五年來整體的頻寬需求大致上成長了 100 倍. 

由於流量的巨量提升，因此下列的所有設施也都必須要因應這些衝擊
  - 提升 Network capacity
  - 資料中心的數量
  -  TE 的路徑數量

3. 此外，在滿足上述條件的改造中，還要確保現有的任何服務不能夠有任何 `downtime`.  

因此，`2018` 的這篇論文主軸就是探討這些問題的本質，並且闡述 `B4` 是如何改造來解決上述的問題並且提供相對應的服務品質與需求。

接下來會去說明到底這`B4`架構的改變中, `google` 學到了什麼概念與想法，透過這些經驗談更可以去理解最後的架構設計的原理。

# Learned
## Flat topology scales poorly and hurts availability
請搭配  [Slides](https://pdfs.semanticscholar.org/63f9/0f7482c186b778ef32b70c877c6d7ec82440.pdf?fbclid=IwAR06jKF1W8bXwga9aRzeVYMEMMXn63vzc6dm1V7OSrJfyquoAUbwNgYmKEE) 的 p.16 - 18 一併觀看

### Saturn
在原先的 `B4` 架構中(p.17)，每個 `site topology` 都成為 `Saturn`, 每個 `Saturn` 中都分成上下兩個部分，下半部分總共有四個 `Saturn chassis`, 而上半部分則是 兩個 或是 四個 的 `Saturn chassis`，其中上下 `Saturn Chassis` 中間溝通的頻寬是 `2.56/5.12 Tbps`. 

為了解決 `Scale` 的問題，`Google` 只能繼續打造更多的 `Datacenter Site` 並且緊鄰原先欲解決 `Scale` 問題的 `Site`. 如圖 `18`.

然而這種愈來愈多的相同地理位置的 `Datacenter Site` 解決方案卻產生三個問題
1. 愈來愈多的 `Datacenter Site` 會增加 `TE (Traffic Engineering)` 計算的成本，導致計算路徑需要的時間更長。 原先 `TE` 的設計就是基於 `Site-Level` 的設計，因此 `Site` 的數量愈多， 計算路徑所需要的時間也相對愈多。原文使用了 `super-linearly` 來描述彼此的關係。 同時 `TE` 計算的時間增加，也會導致當有任何 `data plane` 發生問題時修復所需要的時間
2. 隨者 `Site` 數量的增加，實際上也對底下的交換機產生了更大的壓力，因為交換機內部的傳輸規則表大小是有上限的。
3. 最重要的問題就是，因為`相鄰地區`的`site`大幅度的增加`Capacity`的計算與規劃，對於應用程式開發者來說也產生的很大的困惑。基於 `Site-Level` 的設計下，卻有很多的 `Site` 是要服務相同地區的使用者，但是彼此又非常接近。

在原先的 `B4` 設計中， 所謂的 `Capacity` 計算與規劃主要是用來處理 `site to site` 之間的 `WAN` 頻寬計算

為了解決 `Scale` 的問題，同時又要能夠處理這些延伸出來的挑戰，`Google` 最後提出了全新的 `B4` 架構 `Jumpgate` (p.19 - 20).

### Jumpgate
`Jumpgate` 的整體網路架構不在如同之前 `Saturn` 的扁平，而是採用了階層式的架構.  首先 `site` 是由一個叫做 `supernode` 的基本單位所組成的 (p.19). 每個 `site` 內會包含多組 `supoernode`, 同時這些 `supernode` 會互相連結組出一個 `full mesh` 的網路拓墣來支撐整個 `site`.

`Supernode` 本身則是一個 `兩階層的` Clos 網路拓墣，這部分你可以在(p.19)看到更詳細的圖文說明。

`Google` 說明在基於這種階層式架構下的實驗顯示了其有三個優點
1. Scalability, 可以透過水平擴增的方式增加 `supernode` 即可以增加該 `Site` 節點的能力，而不用繼續增加 `site` 的數量，可以避免讓 `TE` 產生更多的計算問題。
2. Availability, 透過垂直擴增的方式去逐步的更新 `supernode` 就可以避免當前的傳輸服務中斷而被影響。


## Solving capacity asymmetry problem in hierarchical topology
雖然採用了 `階層式架構` 帶來了不少好處，但是經過 `Google` 觀察到 `階層式架構` 對 `TE` 的計算實際上也帶來了不少的問題。
最簡單的範例就是當整個網路實體架構中有任何因為 `設備維護`,`操作` 等導致了當前有任何 `data plane` 裝置不穩定，這些會影響整體連結的頻寬，連帶影響最終的 `Capacity` 的計算，最後會影響整體網路流量的分配。 
這個情形就稱為 `Capacity Asymmetry`, 流量的不對等。
接下來透過 `(p22-24)` 來解釋這個問題到底實際上會產生什麼樣的影響。

在觀看`(p22,p23)` 以前要先強化一個觀念， `TE` 是基於 `Site to Site` 的流量去進行計算的.

首先，先看一下最完美的模型`(p22)`
在這個模型之中我們總共有
1. 三個 `site`, 其中每個 `site` 都各自擁有四個 `supernode`.
2. 每個 `site` 跟 `site` 之間的 `supernode` 也都各自擁有四條連線，其中每個 link 的權重都是 `1`
3. 在此模型下，我們可以計算出每個 `site` 之間的 `capacity` 是 16(4x4), 有四個 supernode, 且每個node有四條連線.
4. 根據 `capacity` 的計算，可以知道這些 `site` 彼此間的最大流量是 `16`。
5. 根據上述的資訊, `TE` 最後分配流量的時後，會分配最多 `16` 的流量到這些 `site` 來傳輸。


接下來考慮到假設遇到一些硬體設備發生故障時的情形。假設第一個 `Site` 裡面的第四個 `Supernode` 發生了一些問題，導致對外連線從本來的 `4` 條變成了 `2` 條。
因此在計算 `capacity` 容量時，會因為這個節點擁有最小的頻寬，也就是所謂的頻頸點。 所以最後計算整體的 `site to site` 之間的頻寬就不會是 16(4x4), 而是 8(4x2). 因此最後透過 `TE` 分配流量時最多只會分配 `8` 單位的流量到第一個 `site`.

實際上當前總共有 14(4x3 + 2) 單位的頻寬可以用，但是因為 `TE` 算法的關係最後只能傳輸 `8` 單位，因此就會有所謂的頻寬浪費(43%)

為了解決這個問題， `Google` 最後引入了兩個新的概念來，分別是 `Sidelink` 以及 `Supernode-Level TE`.

### Sidelink
`sidelink` 的含義就是在同一個 `site` 內的 `supernode` 之間加上一條額外的 `link`. 想法很單純，如果有任何一個 `supernode` 本身對外的頻寬流量是瓶頸的話，那就將該多出來的流量導向其他的 `supernode` 去幫忙轉發處理。

所以根據 `(p.26)` 的圖表，我們可以這樣解讀，現在該 `site` 中的 `supernode` 可以互相轉發流量，所以我們的 `TE` 最後還是可以傳送高達 `14` 單位的流量到第一個 `site` 裡面。
但是如果有超過 `2` 以上的單位流量傳輸到第四個 `supernode` 的話，因為該 `supernode` 本身只有 `2` 的對外流量，因此透過 `sidelink` 將多出來得流量都轉發到其他的 `supernode` 去處理，就能夠盡可能地利用所有 `site to site` 之間的傳輸頻寬。

上述的敘述中有一個沒有描述清楚的東西就是 `TE` 現在要如何去計算 `sidelink` 的容量?

因此 `google` 就提出了不同於之前的 `site-level TE` 的新算法,`supernode-level TE`.

### Supernode-Level TE
為了在`階層式架構中` 能夠正確的處理 `Capacity Asymmetry`，勢必要想出一套支援 `Supernode-level` 的負載平衡演算法來達到最大的 `Site-Level` 頻寬使用率。 此外， `Google` 希望這個新的算法/機制要能夠有下列的特性
1. 當網路架構發生問題狀況時，能夠更快收斂避免當前網路狀況不通的空窗期
2. 基於有限的硬體交換機傳輸規則表大小下能夠更有效率且更高速的去轉發封包。
    - 當 super node 過多的時候，數量就會暴增，這意味整個路徑的計算就會更加的困難與麻煩
3. 最重要的是這個新的機制與算法最多只能使用到一層的封包封裝
    - 太多層的封裝會導致封包處理效率不佳

首先 `Google` 嘗試過基於完全的 `supernode-level` 去進行 `TE` 的計算，結果遇到了一些問題
1. 基於 `supernode-level` 的傳輸，由於跨`site`,因此本身也需要導入 `IP-in-IP` 的封裝.
2. 這個方法有高效能，但是卻沒有辦法有好的收斂與計算時間，整體的時間卻是之前的 188倍， 同時這個方法也需要更好的硬體交換機支援(更大的傳輸規則表大小)

在論文中，`google` 提到了第二個採用的方式，就是 `site-level TE` 加上 `supernode-level` 最短路徑的結合，文中提到這個方式帶來的特性
1. 只需要一次的封包封裝就可以完成 `site to site` 的傳輸
2. 可達成 `scalability`, 畢竟 `shortest path` 的計算成本比較低
3. 但是 `shortest path routing` 也會帶來不少問題，譬如 `sidelink` 的權重設計可能就會導致該 `sidelink` 完全不會被用到，最後就回到最原始的 `Capacity Asymmetry` 問題。


## Hierachical TE
為了解決這一系列的問題，最後產生了 `階層式` 的 `TE` 架構。
在這個架構中分成了
1. Flow Group (FG)
2. Tunnel Group (TG)
3. Tunnel Split Group (TSG)
4. Switch Split Group (SSG)

這部分的最後架構以及算法實在有點多，有興趣的人請自行參閱 [B4 and after: managing hierarchy, partitioning, and asymmetry for availability and scale in google's software-defined WAN](https://dl.acm.org/citation.cfm?id=3230545&fbclid=IwAR3JnZf8yk2Ku8JXutQZSsAwm8Koca2ZSlInGI7JOJb9P3rAu91EWHai2c0) 中的 section `4.2` 開始學習全貌。

這部分之後我會再額外開一篇文章來介紹其算法與實現。


## Efficient switch rule management
由於每個硬體交換機本身所支持的轉發規則表格大小是有限度的，因此要如何透過這些有限度的資源來滿足之前提供對應的轉發能力來滿足一切需求就是一大挑戰
{% note info %}
由於先前的 `TE` 架構沒有完全說明清除，因此此章節在探討規則的時候會有部分沒有辦法釐清其原理，不過就算如此，我們還是可以學習 `Google` 針對這類型問題時思考問題的脈絡與方向
{% endnote %}

根據前述的討論，透過了 `階層式` 的 `TE` 設計雖然能夠解決不少的問題，但是其需要的轉發規則數量也急速上升，超過了硬體交換機當前的限制。

為了解決這個問題， `Google` 提出了兩個方式來解決問題
1. 將 `flow matching` 相關的規則拆給給兩個 `switch pipeline table` 來處理，藉由繼續這種`階層式`的兩階段比對規則，`google`發現到可以支援的 `site` 數量提升到 `60倍`，意味還有很多的空間供擴充。
2. 接下來將 `path` 的比對概念也拆開，在一個 `Clos fabric` 的網路架構中，將 `Path` 路由相關的概念分別實作於 `Edge Switch` 以及 `Backend Switch` 兩個階段處理就能夠完全的處理之前所設計 `階層式` TE 所需要的一切路由規則。
    - 採用這個解法雖然彈性且強大，但是實際上會降低整體的流量大概 `6%`, `google` 認為是一個可接受的損失，利大於弊。ㄌ

接下來針對 `p39 - p44` 進行一個比較細緻的解說
首先 `p39` 描述的是最基本原始的狀態，每個交換機內部的 `pipeline table` 包含了三個狀態，分別是 `ACL`, `ECMP` 以及 `Encp`.
用來比對相關的規則，尋找路由規則最後進行封裝。

### ACL

但是此種做法並沒有辦法支撐新架構 `TE` 所帶來的大量規則。
因此在 `p40` 這邊描述了第一個問題，就是到底 `ACL` 路由表的規則數量到底有多少

Size(ACL) ≥ (#Sites ✕ #PrefixesPerSite ✕ #ServiceClasses)

根據 `p40` 的統計資料顯示， `ACL` 的規則大概是 `3k` 也就是三千條左右，然而總共需要的比對規則數量則是 `Site` x `PrefixesPerSite` x  `ServiceClasses`.
1. 目前有 33 個節點
2. 根據論文顯示，平均的數量大概是16 左右
3. 目前有 6 種類別 `SC0 - SC5`
在這個情況下，三個數值得到的結果大概就是 `3k`, 這意味者已經到達了極限，幾乎沒有辦法進行更大的擴充，不論是 `site`, `IP prefix` 或是 `Service` 的種類。

如同上述所說，一旦將 `flow table(ACL)`拆成兩個部分，分別是 `VFP` 以及 `Per-VRF LPM` 兩個規則進行處理，最後達到的效果就是可以提升將近 60倍 數量的規則數.(意味者在IP跟服務種類不變的情況下，可以擴充到1900多個節點左右)
概念如下
1. 將 `cluster prefix` 符合相關的規則全部移動到 `LPM (Longest Prefex Match)` 表格中。 
{% note info %}
在交換機中，不同表格的大小不一定，根據論文顯示， `LPM` 表格的數量上限遠遠比 `ACL` 還要來得高。
{% endnote %}
2. 由於 `LPM` 表格中沒有辦法針對 `DSCP` 的比對(DSCP用來代表不同的服務), 但是 `LPM` 可以比對 `VRF (Virtual Routing Forwarding)` 的標籤，因此決定透過 `VRF` 與 `DSCP` 進行一個比對的關係。
3. 一開始則使用 `VFP(Virtual Forwarding Plane)` 表格來進行比對，在這個表格中則會透過 `DSCP` 的比對並且設定特定的 `VRF` 標籤，供後續的 `LPM` 表格識別其為特定的應用服務

### Traffic Hashing
接下來的 `p42` 要描述的則是 `ECMP` 表格比對的大小問題。
相關的數字該投影片中都已經描述了，結論來說就是按照目前的設計，需要大概 `198K` 的大小，但是實際上只有 `16K` 可以使用。 這龐大的差距最後的解決概念就是如同 `p43` 所描述，將不同概念的比對放到 `Clos Fabric` 網路中不同階層去處理。
這邊的概念必須要完整理解 `TSG/GS/SSG/TG` 等各種不同階層的 `TG` 才更好理解。

# Summary

整篇論文大概 `13` 左右，搭配該投影片來閱讀大概可以理解其遇到的問題以及大概上解決的想法。
然而一些更細部的技術探討，主要是 `TG` 整個演算法的處理以及決策順序都必須要反覆的閱讀該論文並且理解其 `Pseudo Code` 才能領悟。
此外論文中也有一些數據的比較，這部分在投影片中也只有稍微的一些比對，並沒有如論文裡面的詳細。

我認為有時間的話，還是值得將該篇論文好好地仔細咀嚼去學習一下整體背後的脈絡

最後直接引用投影片中的比較表來一個總結

| Before |  After|
| -------- | -------- |
| Copy network with 99% availability     | High-available network with 99.99% availability     | 
| Inter-DC WAN with moderate number of sites   | 100x more traffic, 60x more tunnels     | 
| Saturn: flat site topology & per-site domain TE controller  | Jumpgate: hierarchical topology & granular TE control domain    | 
| Site-level tunneling  | Site-level tunneling in conjunction with supernode-level TE (“Tunnel Split Group”)     | 
| Tunnel splits implemented at ingress switches   | Multi-stage hashing across switches in Clos supernode     | 


# Reference
1. [B4 and after: managing hierarchy, partitioning, and asymmetry for availability and scale in google's software-defined WAN](https://dl.acm.org/citation.cfm?id=3230545&fbclid=IwAR3JnZf8yk2Ku8JXutQZSsAwm8Koca2ZSlInGI7JOJb9P3rAu91EWHai2c0)
2. [Slides](https://pdfs.semanticscholar.org/63f9/0f7482c186b778ef32b70c877c6d7ec82440.pdf?fbclid=IwAR06jKF1W8bXwga9aRzeVYMEMMXn63vzc6dm1V7OSrJfyquoAUbwNgYmKEE)

