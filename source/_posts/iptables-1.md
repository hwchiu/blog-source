---
title: 初探 IPTABLES 流動之路 - 以 Docker 為範例
tags:
  - Network
  - Linux
  - Iptables
  - Container
keywords: 'Kubernetes,Network,Linux,Ubuntu'
description: >-
  本文透過 IPTABLES/EBTABLES 的規則與 Docker Container
  的環境來觀察不同情況下的封包流向，主要是幫線上議程進行的重點整理，主要內容都在影片中。
abbrlink: 39531
date: 2020-05-09 19:01:27
---



本文主要是針對 [SDN x Cloud Native Meetup - Webinar 海外篇 #1
](https://www.meetup.com/CloudNative-Taiwan/events/269343753/) 這場線上演講進行重點整理，詳細內容請參閱 [Youtube紀錄](https://www.youtube.com/watch?v=y4e_B6PdX8A)


{% youtube y4e_B6PdX8A %}


# Introduction

本次演講主要是探討封包基於 Docker Container 的預設環境下，不同走向實際上對應到 iptables/ebtables 的實際流向。
演講中主要針對三個方向進行探討，分別是
1. Host to Container
2. Container to Container
3. Container to WAN

每個方向都包含了雙向來回，譬如
Host to Container 其實也包含了 Container to Host
Container to WAN 也包含了 WAN to Containe
主要差別在於誰發起了連線，這中間差異不大 (WAN to Container 可能會有 DNAT)

下圖是這次演講最主要的圖片，接下來會透過實際觀察的方式，一步一步的建立起這張圖片
![](https://i.imgur.com/tAXRdHs.png)

# Environment
本影片的測試環境可透過 Vagrant 的方式還原，相關資源都放在 [hwchiu
/
network-study](https://github.com/hwchiu/network-study/tree/master/iptables/iptables_ebtables) 這個專案內，可以透過 `vagrant up` 的方式將整個環境建立起來

當整個 OS 都準備完畢之後，接下來會採用下列的環境來啟動兩個 Container 並且使用預設的 Linux Bridge 模式。
![](https://i.imgur.com/W6gqCCG.png)

接下來所有的操作都會基於這兩個 Container 的互動來觀察封包流向

# Basic Concept
不論是 iptables, ebtables 或是本文沒有提到的 arptables, 其架構都可以用下列這張圖片來解釋
![](https://i.imgur.com/ujpKTQb.png)

## User/Kernel

所有你可以直接在系統上操作的 `iptables-` 系列工具 (ebtables,arptables雷同)全部都是 `User Space` 的工具，其功能都是用來管理 `規則`, 但是規則真正運行被觸發的時機點都是在 `Kernel` 內。

實際上， `iptables` 會透過 `getsockopt/setsockopt` 等 IPC 方式與 Kernel 進行溝通，不論是讀取當前的規則，或是寫入新規則，這邊也有一個重要的概念就是 **規則** 本身不會被儲存，所以當機器重新開機的時候， **kernel** 內的規則就會全部消失，需要仰賴 **userspace** 的工具重新寫入規則到 **kernel** 內，

上述的架構其實也會讓整個 **iptables** 的觀察與管理變得相對困難，因為大部分情況下我們都是使用 **iptables-*** 系列工具來進行觀察與管理，而實際上封包到底怎麼流動，被哪些規則給丟棄，被哪些規則給修改，一切都是在 **kernel** 內進行。
這意味者我們必須要相信 **iptablse** 與 **kernel** 的溝通是沒有問題的，不然單純依靠 **userspace** 的工具來觀察結果其實是會有一些不確定性。

但是如果要觀察這一切的訊息都要去改 Kernel 來幫助除錯，這方面的工與精力也花費太大，所以一般情況下都還是基於 **iptables** 的規則來解讀封包與 iptables 之間的關係。

## 規則組成

大抵上， iptables/ebtables/arptables 的規則組成是有一訂的規範，一個使用範例如下
![](https://i.imgur.com/Ki0OnyE.png)

這個規範中可以分成四個部分
1. Chain: 最簡單的想法就是封包發生的時間點，譬如上述的 **OUTPUT**, **INPUT**
2. Table: 一群相同規則的集合體，譬如 **nat**, **filter**, 該 **table** 內的規則都有類似的目的
4. Match: 符合的條件，可以想成該規則要被觸發的條件，譬如上述的 **! -d 127.0.0.0/8 -m addrtype --dst-tppe LOCAL**, 這部分可以是 **iptables** 內建的基本條件，或是透過 **-m** 來動態載入的其他的 module
5. Target: 當規則符合條件後，要做什麼事情，譬如上述的 **-j DOCKER, --log-level debug**
, 如同 **Match** 一樣，有內建的 **Target** 外也可以動態載入其他 module 來處理。

所以今天如果要寫入一個 iptables規則，思路大概是
我想要撰寫一個規則，這個規則會在 `什麼時間點` 被觸發，`什麼樣的封包` 符合條件，最後要`執行什麼動作`.
而根據動作的類型再把這個規則放到對應的 `Table` 裡面。


## 觀察方式

有了上面的概念後，本文就會使用 **LOG** 概念的 **Target** 來觀察封包流向，整個規則的含義就是
在**什麼時間點**,**針對我們想要觀察的流向封包**,**輸出相關資訊**。
藉由這些資訊，我們可以組合出封包於 iptables/ebtalbes/arptables 內的流向
**注意，這邊只能做到 iptables/ebtables/arptables 內的流向，其餘更細部的 network stack 處理則沒有辦法**

# EBTABLES
EBTABLES 相對於 **IPTABLES** 來說比較陌生，主要是其運作的層次更低，基於 **ethernet** 來處理，一般使用情況下，大家都比較依賴 **iptables**，也是因為透過 **IP** 的描述方式會比使用 **MAC Address** 來得更佳容易記憶與管理。

即管如此，**ebtables** 的存在還是不可忽視，也許某一天你的應用場景就會需要使用到 **ebtables** 來管理

## Chain
EBTABLES 內總共有六條 **Chain**，也就是六個時機點
1. INPUT:
根據 **Forwarding table**, 該封包要送到 **Linux Bridge** 前
2. FORWARD:
根據 **Forwarding table**，該封包要被轉發到其他連接埠前 (不能是 Linux Bridge 本身，否則會走 INPUT)
3. OUTPUT:
本地產生的封包，最後目的是 **Linux Bridge** 底下的連接埠，基本上與 **FORWARD** 非常相似，只是一個來源是其他人，一個是自己機器本身
4. PRE-ROUTING:
我看過一些評論，跟我的想法滿類似的就是這邊準確的說應該是 **PRE-FORWARDING**, 因為這個層級我們不探討 **IP**，不會用 **Routing** 這個字，但是理論與實作本來就會有所差異，與 **IPTABLES** 共用相同的結構在實作上會比較簡單些。
這個時間點主要是用在剛剛從 `Linux Bridge` 相關連接埠收到封包後，還沒有透過 **Forwarding table** 決定目標前，這也是為什麼稱為 **PRE-** 的原因，主要都是用來修改封包內容

題外話：因為還沒有被 **Forwarding Table** 決定怎麼轉送，所以通常這時候都可以修改封包的目的地

5. POST-ROUTING:
基本上跟 **PRE-ROUTING** 是雷同的，概念改成 **封包要從網卡出去前** 會觸發的時間點，也會用來修改封包

題外話：因為還已經被 **Forwarding Table** 決定怎麼轉送，通常這時候不可以修改封包的目的地，但是可以修改封包的來源

7. BROUTING:
這是一個 ebtalbes 獨有的點，非常少用，觸發時間點非常早，封包收到後就會先進入到這邊去處理
這個時間點能做的事情只有 **封包要不要直接送到 Layer3 去處理** 的判斷。

以上就是 **ebtalbe** 的六個時機點，接下來看看有哪些 **table** 最後如何組合再一起

## Tables
Table方面則是會有三個
1. Filter
用來過濾封包，可以決定要不要丟棄或是通過
2. NAT
針對 PRE-ROUTING/POST-ROUTING 等時機點使用，去修改封包的 MAC Address
3. Brout
這個是非常特別，就針對 BROUTING 這個 chain 去使用而已

每個 Table 都有自己搭配的 Chain, 所有的規則都定義於 **Linux Kernel** 內，初始化這些 **Table** 的時候就會去決定有哪些 **Chain** 可以與之匹配。

## Workflow
將上述的 **Chain** 與 **Table** 結合後會得到這張圖片
![](https://i.imgur.com/didIHNQ.png)

圖片中先忽略所有 **iptables** 的處理，專心觀察 **ebtables** 的架構，圖中的 **Bridging Decision** 也就是本文上述提的 **Forwarding table**, 用來決定方向。

## Lab
接下來要透過實際操作來驗證上述的行為，會使用類似於下面用法的規則來顯示資訊
![](https://i.imgur.com/avlfprm.png)
這樣當封包經過時就會將資訊吐出，可以透過 **dmesg** 來觀察

完整規則如下
![](https://i.imgur.com/W4HzYOi.png)

### Container to Container
Contaienr to Contaienr 的方式非常簡單，就直接讓兩個 Container 透過 **ICMP request/reply** 來產生流量即可

```bash=
sudo docker exec netutils ping 172.18.0.2 -c1
```

![](https://i.imgur.com/hD5y2GM.png)
最後觀察 **kernel log** 會得到類似下列的輸出

![](https://i.imgur.com/R5bEplY.png)
將這些輸出結果與前面的流程圖合併後，可以得到下列的資訊
![](https://i.imgur.com/DwPgD3L.png)

基本上 Container to Container 就只會經過這四個點

### Host to Container
Host to Container 也是使用 **ICMP Request/Reply** 來處理，不過是由 **host** 本身發起，所以概念如下
![](https://i.imgur.com/CpIONzF.png)
```bash=
sudo ping 172.18.0.2 -c1
```

觀察 **kernel log**會得到類似下列輸出
![](https://i.imgur.com/9ExPHOr.png)
整理後會得到類似這種走向
![](https://i.imgur.com/KWaemyK.png)

可以看到 **Host to Contaienr** 則會走 **output** 出去，而 **Container to Host** 最後則是在 **Bridging Decision** 決策後走 **INPUT** 上去。

更多詳細的操作過程可以觀看影片，裡面有實際的 Demo 以及相關的操作。

# IPTABLES

IPTABLES 概念非常雷同，有四個 **Table**, 5個 **Chain**，此外還有 **Conntrack(Connnection Track)** 在輔佐幫忙。

這邊我們專注於 **IPTABLES** 本身， **Conntrack** 的概念就不紀錄太多，影片中有一些段落再介紹其概念與影響。

## Chain
1. INPUT
2. FORWARD
3. OUTPUT
4. PREROUTING
5. POSTROUTING

這邊的概念跟 **EBTALBE** 完全一樣，只是 **全部的 Forwarding Table** 都要換成基於 **IP** 查詢的 **Routing Table**，並且沒有了 **BROUTING** 這個點。

## Table
除了原有了 **Filter** 以及 **NAT** 之外，多出了 **RAW** 以及 **MANGLE** 兩張 **Table**.

1. RAW
RAW 這個 Table 本身被呼叫的順序就很早，基本是被 **Conntrack** 處理前會呼叫的 Table
2. Mangel
這個可以用來進行一些修改封包的內容，譬如 mark 等之類的資訊

最常用的還是 **filter** 以及 **NAT**。

## Workflow

iptables 本身的流向簡化版本如下

![](https://i.imgur.com/wQoCYtx.png)

概念跟 **ebtables** 非常像，這邊透過 **Routing** 來決定封包走向。



由於前面有 **ebtables** 的經驗，我們先將兩者合併的圖片展示，接下來再透過觀察的方式驗證這張圖片中的封包流向
![](https://i.imgur.com/rTmTiJz.png)

## Lab

觀察的方式跟 **ebtable** 一樣，透過 **LOG** 的方式來觀察封包，譬如
```bash=
iptables -t mangle -I PREROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix '/iptable/mangle-PREROUTE' --log-level debug
```

完整的規則如下
```bash=
insert() {
    iptables -t raw -I PREROUTING -p icmp -j LOG --log-prefix 'iptable/raw-PREROUTE' --log-level debug
    iptables -t mangle -I PREROUTING -p icmp -j LOG --log-prefix 'iptable/mangle-PREROUTE' --log-level debug
    iptables -t nat -I PREROUTING -p icmp -j LOG --log-prefix 'iptable/nat-PREROUTE' --log-level debug

    iptables -t mangle -I FORWARD -p icmp -j LOG --log-prefix 'iptable/mangle-FORWARD' --log-level debug
    iptables -t filter -I FORWARD -p icmp -j LOG --log-prefix 'iptable/filter-FORWARD' --log-level debug

    iptables -t mangle -I INPUT -p icmp -j LOG --log-prefix 'iptable/mangle-INPUT' --log-level debug
    iptables -t filter -I INPUT -p icmp -j LOG --log-prefix 'iptable/filter-INPUT' --log-level debug


    iptables -t raw -I OUTPUT -p icmp -j LOG --log-prefix 'iptable/raw-OUTPUT' --log-level debug
    iptables -t mangle -I OUTPUT -p icmp -j LOG --log-prefix 'iptable/mangle-OUTPUT' --log-level debug
    iptables -t nat -I OUTPUT -p icmp -j LOG --log-prefix 'iptable/nat-OUTPUT' --log-level debug
    iptables -t filter -I OUTPUT -p icmp -j LOG --log-prefix 'iptable/filter-OUTPUT' --log-level debug

    iptables -t mangle -I POSTROUTING -p icmp -j LOG --log-prefix 'iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -I POSTROUTING -p icmp -j LOG --log-prefix 'iptable/nat-POSTROUTE' --log-level debug
 }
 ```


ebtables 的規則也一併保留一起使用，就可以觀察 iptables+ebtables 的流向
### Contaienr to Container
指令如下
```bash=
sudo docker exec netutils ping 172.18.0.2 -c 1
```
下面兩張圖就是去回流向，這邊有兩個點要注意
![](https://i.imgur.com/0fCdsSy.png)
![](https://i.imgur.com/OHGBMsJ.png)

1. 回來的封包不會進入到 **NAT** table, 主要是這些封包被 **Conntion Track(Conntrack)** 給處理過，接下來都不會進入 **NAT** 處理。
2. `ebtables` 內會偷偷呼叫 **iptables** 來進行處理，這部分是個動態開關，可以透過
`/proc/sys/net/bridge/bridge-nf-call-iptables
` 來告訴 **kernel** 要不要偷偷呼叫 **iptables**。

這樣的好處就是對於 **Contaienr to Container** 的封包流向，可以使用 **iptables** 來管理，不然就單純只能使用 **ebtables** 來使用。


### Host to Container

執行指令
```bash=
ping 172.18.0.2 -c 1
```

相關的 Log 請參閱影片結果，底下就紀錄最後結果

Host to Contaienr
![](https://i.imgur.com/OUf3KcS.png)

Container to Host
![](https://i.imgur.com/PiMQ4uZ.png)



### Container to WAN
執行指令
```bash=
sudo docker exec netutils ping 8.8.8.8 -c 1
```
相關的 Log 請參閱影片結果，底下就紀錄最後結果

Container to WAN
![](https://i.imgur.com/uLcRpGW.png)

WAN to Container
![](https://i.imgur.com/pWgDbjA.png)

# Summary

經過這次的觀察，大致上的結論是
1. 網路很難，除錯很麻煩
2. IPTABLES 很難，除錯很麻煩
3. Linux Kernel 很複雜，除錯很麻煩

如果除錯這麼難，那網路出問題到底該怎麼辦
1. 網路基本概念的理解
2. Linux 網路運作的理解
3. TCPDUMP 等相關工具的使用
4. 檢查 IPTABLES/EBTABLES/ARPTABLES 等規則
5. 檢查 Routing Table/ARP Tables/Forwarding Tables 的紀錄
6. 相關資訊加上經驗整合，最後歸納出一套封包的走向，並且去驗證

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