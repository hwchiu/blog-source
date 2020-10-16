---
title: '[netfilter] Dig Into Docker Bridge Network By iptables/ebtables'
keywords: 'Docker,Network,iptables,Bridge,HowTo,netfilter'
abbrlink: 34860
date: 2018-09-18 10:49:21
tags:
  - Network
  - Netfilter
  - Linux
  - Docker
description: 本文透過對 iptables/ebtables 來設定相對應的規則，藉由這些規則來觀察在 Docker Bridge Network 的網路設定下，不同情境的網路傳遞實際上會受到哪些 iptables/ebtables 規則的影響。這些情境包含了常見的用法，譬如容器與容器之間同網段的傳輸，宿主機透過容器IP位址直接連線，甚至是外部網路透過 docker run -p xxx.xxx 的規則來接觸到內部容器。這些不同情境的網路連線牽扯到關於 Layer3 路由，Layer2 橋接 等不同方式的處理，因此在 iptables/ebtables 都會有不同的走向。只要能夠更佳的熟悉 iptables/ebtables 的用法與規則，未來有需要親自設定相關規則時，都能夠更精準且安全的去達到想要的目的，減少盲目猜測的時間與花費。

---

## Preface
這次想要跟大家慢慢介紹的就是 `iptables` 這個常見也常用的工具。
網路上其實已經可以搜尋到非常多關於 `iptables` 相關的文章。
不論是基本介紹，或是一些相關用法，其實都有滿多的資源可以學習，不過我認為這些文章都散落各地，所以想要整理一下這些資訊並且統整起來做一個一系列的`iptables` 文章。

這個系列文的內容大致上如下
1. iptables/ebtables 的基本架構介紹，包含下列各種組成的概念
    - Target/Chain/Table/Match
2. 透過 `docker` 預設網路`Bridge`的情況來解釋，容器與外界網路，容器與容器彼此之間的網路傳輸，實際上再 `iptables` 中到底會怎麼運作，如果想要處理這些封包，該怎麼設定相關
規則。
3. 介紹相關 `iptables` 常見的使用問題
4. 最後則是會跟大家介紹，如何自己手動撰寫一個 `iptables` 擴充模組，讓你的`iptables`擁有獨一無二的功能


本文延續前一篇 `ebtables` 的介紹，將使用相同的概念來闡述 `iptables(ipv4)` 的概念，包含了 `Tarble/Chain/Match/Target` 等功能。

相關系列文章
- [[netfilter] Introduction to ebtables](https://www.hwchiu.com/netfilter-eiptables-i.html)
- [[netfilter] Introduction to iptables](https://www.hwchiu.com/netfilter-eiptables-ii.html)

## Introduction
本文是結合前述兩篇理論文章後的實戰文，要透過對 `iptables/ebtables` 的操作來實際觀察封包於不同的情境之中傳輸實際上會經過哪些 `iptables/ebtables` 的控管。

### Software Requirement
在實際觀察前，我們需要先建立好一個容易測試的環境，我自己測試的環境如下
- Linux: 4.4.0-128-generic
- Ubuntu: Ubuntu 16.04.4 LTS
- Docker: 17.06.2-ce

整個測試用的所有程式以及相關腳本都可以在 [iptables experience](https://github.com/hwchiu/iptablesExample/tree/master/docker) 這邊找到

### Environment
本文所有的測試情境都會基於下圖的環境。左邊是以上帝視角的視野來觀察整個測試環境，右邊則是採用 `User-Space/Kernel-Space` 此角度來觀察測試環境

![Imgur](https://i.imgur.com/Zi8190W.png)

首先會先準備兩個 Container 容器，這兩個容器分別扮演 `Nginx Server` 以及 `Ping Clinet` 的角色。

此外，主機上面本身也要擁有 `Ping` 的能力，若沒有的需要進行安裝，否則本文後續的測試會沒有辦法繼續。

最後我們會嘗試進行三個不同類型的封包傳輸，觀察這些封包實際上會受到哪些 `iptables/ebtables` 的影響。
- Container Bash 透過 ping 指令連線到 Container Nginx
- 外網連到 Container 內的 Nginx
- 本機透過 `ping` 連到 Container 內的 Nginx

### Setup Docker Containets
首先，確認主機本身已經有安裝 Docker 相關的服務，接者執行下列程式來運行兩個 Docker 容器於主機上。

```bash=
#!/bin/bash
docker rm -f nginx
docker rm -f netutils

docker run -d -p 5566:80 --name nginx nginx
docker run -d --name netutils hwchiu/netutils
```

同時為了確保能夠正常運作所有指令，可執行下列指定將相關指令安裝起來
```bash=
apt-get install -y ebtables iputils-ping
```

此外，為了詳細的觀察 `iptables/ebtables` 對連線封包的傳輸，我們要使用 `Log` 相關的 `Target` 來操作這些，最後這些 `Log` 的資訊都會從 `Kernel` 打印出來，
我們可以透過 `Dmesg` 的方式去觀察這些封包。

基本上在 iptables 是採用 -j LOG 的方式來處理，然而在 ebtables 則是直接採用 --log 這種原生的方式來處理，其隱性的使用 -j CONTINUE 去繼續處理封包。

實際上我們可以用 `dmesg -c` 的方式，每次呼叫都只會顯示新出現的部分，這樣會更容易幫助我們觀察封包

在 `Log` 的指令中，透過 `--log-prefix` 的方式去列印更多的資訊，可以幫助我們更好觀察。

範例指令
```bash=
iptables -t mangle -I PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-PREROUTE' --log-level debug

ebtables -t filter -I INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
```

## Container To Container
在這個測試情境中，我們要觀察容器與容器之間的傳輸，如下圖。

![Imgur](https://i.imgur.com/pDtDXsP.png)

在這邊我們要從一個容器使用 `ping -c1` 去傳送一個封包到另外一個容器。

然後藉由 `dmesg` 這個指令來觀察果。

由於我自己所下的 `iptables/ebtables` 的規則非常簡單，所以`強烈建議`系統上不要有任何其他的容器正在有任何的網路傳輸，否則 `Kernel` 輸出會讓你很難理解每個訊息的先後順序。

### Setup ebtables
我用來建立跟刪除 `ebtables` 規則的腳本如下
```bash=
#!/bin/bash

insert() {
    ebtables -t broute -I BROUTING --log --log-prefix 'ctc/ebtable/broute-BROUTING' --log-level debug
    ebtables -t nat -I PREROUTING --log --log-prefix 'ctc/ebtable/nat-PREROUTE' --log-level debug
    ebtables -t nat -I POSTROUTING --log --log-prefix 'ctc/ebtable/nat-POSTROUTE' --log-level debug
    ebtables -t filter -I INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
    ebtables -t filter -I OUTPUT --log --log-prefix 'ctc/ebtable/filter-output' --log-level debug
    ebtables -t filter -I FORWARD --log --log-prefix 'ctc/ebtable/filter-forward' --log-level debug
}

delete() {
    ebtables -t broute -D BROUTING --log --log-prefix 'ctc/ebtable/broute-BROUTING' --log-level debug
    ebtables -t nat -D PREROUTING --log --log-prefix 'ctc/ebtable/nat-PREROUTE' --log-level debug
    ebtables -t nat -D POSTROUTING --log --log-prefix 'ctc/ebtable/nat-POSTROUTE' --log-level debug
    ebtables -t filter -D INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
    ebtables -t filter -D OUTPUT --log --log-prefix 'ctc/ebtable/filter-output' --log-level debug
    ebtables -t filter -D FORWARD --log --log-prefix 'ctc/ebtable/filter-forward' --log-level debug
}

check() {
    count=`ebtables-save | grep ctc| wc -l`
    if [ "$count" == "0" ]; then
        echo "Delete Success"
    else
        echo "Delete Fail, Use the ebtables-save to check what rules still exist"
    fi
}

if [ "$1" == "d" ]; then
delete
check
else
insert
fi
```

執行裡面的 `insert` 函式就可以對 `ebtables` 的所有 `Table/Chain` 組合都寫一條規則，注意的是我採用的是 `-I xxx` 意味者將該規則放到第一條，避免我們的規則因為其他的規則沒有被順利執行。

實驗結束時可以透過 `delete` 函式去移除相關的規則

### Setup iptables
`iptables` 的概念非常雷同，但是因為系統本身有太多網路流量在傳輸，所以我有特別設定 `-s 172.18.0.0/16 -d 172.18.0.0/16` 這規則來確保只有封包的來源與目的地都是屬於容器之間的才會去紀錄。

此外，我特別下了一條 `mangle` 的規則是因為 `nat` 在 `PREROUTING/POSTROUTING` 會因為 `conntrack` 的幫忙導致看不出來有被執行多次，所以特別多用一個 `mangle` 來幫忙釐清。

```bash=
#!/bin/bash

insert() {
    iptables -t mangle -I PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t mangle -I POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -I PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-PREROUTE' --log-level debug
    iptables -t nat -I POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-POSTROUTE' --log-level debug
    iptables -t filter -I INPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-input' --log-level debug
    iptables -t filter -I OUTPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-output' --log-level debug
    iptables -t filter -I FORWARD -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-forward' --log-level debug
}

delete() {
    iptables -t mangle -D PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t mangle -D POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -D PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-PREROUTE' --log-level debug
    iptables -t nat -D POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-POSTROUTE' --log-level debug
    iptables -t filter -D INPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-input' --log-level debug
    iptables -t filter -D OUTPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-output' --log-level debug
    iptables -t filter -D FORWARD -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-forward' --log-level debug
}

check() {
    count=`iptables-save | grep ctc| wc -l`
    if [ "$count" == "0" ]; then
        echo "Delete Success"
    else
        echo "Delete Fail, Use the iptables-save to check what rules still exist"
    fi
}


if [ "$1" == "d" ]; then
delete
check
else
insert
fi
```
### Test
當上述規則的設定完畢後，我們先執行數次 `dmesg -c` 去確保目前沒有任何 `kernel` 所輸出的新訊息。
接者執行下列指令，請先確保(172.18.0.2)是 容器 `nginx` 的 `IP` 地址
```bash=
docker exec  netutils ping 172.18.0.2 -c1
```

接下來馬上執行 `sudo dmesg -ct` 來顯示資料。(透過 -t 只是不想要顯示時間，排版比較好看)

該輸出資料如下，我們將該資料分成兩部分，因為 `ping -c1` 實際上會牽扯到 `ICMP Request` 以及 `ICMP Reply`.

如果你的環境中有看到 proto=0x0806, 這是所謂的 APR 封包，本文暫時不討論 ARP。

```bash=
ctc/ebtable/broute-BROUTING IN=vethd709394 OUT= MAC source = 02:42:ac:12:00:03 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
ctc/ebtable/nat-PREROUTE IN=vethd709394 OUT= MAC source = 02:42:ac:12:00:03 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
ctc/iptable/mangle-PREROUTEIN=docker0 OUT= PHYSIN=vethd709394 MAC=02:42:ac:12:00:02:02:42:ac:12:00:03:08:00 SRC=172.18.0.3 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=7179 DF PROTO=ICMP TYPE=8 CODE=0 ID=8896 SEQ=1
ctc/iptable/nat-PREROUTEIN=docker0 OUT= PHYSIN=vethd709394 MAC=02:42:ac:12:00:02:02:42:ac:12:00:03:08:00 SRC=172.18.0.3 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=7179 DF PROTO=ICMP TYPE=8 CODE=0 ID=8896 SEQ=1
ctc/ebtable/filter-forward IN=vethd709394 OUT=veth5ead5c7 MAC source = 02:42:ac:12:00:03 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
ctc/iptable/filter-forwardIN=docker0 OUT=docker0 PHYSIN=vethd709394 PHYSOUT=veth5ead5c7 MAC=02:42:ac:12:00:02:02:42:ac:12:00:03:08:00 SRC=172.18.0.3 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=7179 DF PROTO=ICMP TYPE=8 CODE=0 ID=8896 SEQ=1
ctc/ebtable/nat-POSTROUTE IN= OUT=veth5ead5c7 MAC source = 02:42:ac:12:00:03 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
ctc/iptable/mangle-POSTROUTEIN= OUT=docker0 PHYSIN=vethd709394 PHYSOUT=veth5ead5c7 SRC=172.18.0.3 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=7179 DF PROTO=ICMP TYPE=8 CODE=0 ID=8896 SEQ=1
ctc/iptable/nat-POSTROUTEIN= OUT=docker0 PHYSIN=vethd709394 PHYSOUT=veth5ead5c7 SRC=172.18.0.3 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=7179 DF PROTO=ICMP TYPE=8 CODE=0 ID=8896 SEQ=1
```

透過我們事先描述好的 `log-prefix`, 我們可以很清楚的觀察到 `iptables/ebtables` 比對的過程。
這些規則我只針對幾個有趣的部分介紹一下
1. 前面封包的`IN=`代表的都是本機上用來將 `Docker`容器與 `Linux Bridge` 連結的 `veth` 虛擬連線。
2. 可以觀察到前面幾個訊息的 `OUT=` 都是空的，這是因為還沒有進行 `Bridge Decision`, 還沒有辦法知道封包到底目標的網卡是誰。
3. 可以看到在 `ebtables` 這邊的 `in=` 都是 `vethxxx` 而 `iptables` 的都是 `docker0`, 這是因為兩者層及不同，關注的點不一樣，實際在上 `iptables` 中可以透過 `physical` 相關的參數拿到 `vethxxxx`.
4. 經過 `FORWARD` 之後，可以觀察到 `IN` 的訊息都不見了，這是因為在 `PREROUTING`  這邊可以進行 `SNAT` 之類的選項，可以改變封包的送端是誰，所以這時候 `IN=` 的資料其實就是一個不確定性，而且也沒那麼重要了。

```bash=
ctc/ebtable/broute-BROUTING IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:ac:12:00:03 proto = 0x0800
ctc/ebtable/nat-PREROUTE IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:ac:12:00:03 proto = 0x0800
ctc/iptable/mangle-PREROUTEIN=docker0 OUT= PHYSIN=veth5ead5c7 MAC=02:42:ac:12:00:03:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.18.0.3 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=59995 PROTO=ICMP TYPE=0 CODE=0 ID=8896 SEQ=1
ctc/ebtable/filter-forward IN=veth5ead5c7 OUT=vethd709394 MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:ac:12:00:03 proto = 0x0800
ctc/iptable/filter-forwardIN=docker0 OUT=docker0 PHYSIN=veth5ead5c7 PHYSOUT=vethd709394 MAC=02:42:ac:12:00:03:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.18.0.3 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=59995 PROTO=ICMP TYPE=0 CODE=0 ID=8896 SEQ=1
ctc/ebtable/nat-POSTROUTE IN= OUT=vethd709394 MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:ac:12:00:03 proto = 0x0800
ctc/iptable/mangle-POSTROUTEIN= OUT=docker0 PHYSIN=veth5ead5c7 PHYSOUT=vethd709394 SRC=172.18.0.2 DST=172.18.0.3 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=59995 PROTO=ICMP TYPE=0 CODE=0 ID=8896 SEQ=1
```

針對 `ICMP Reply` 回傳的部分，我們首先可以觀察到
1. 訊息數量不對稱，少了兩個比對的訊息
2. 少的分別是 `iptable/nat-PREROUTEIN` 以及 `ctc/iptable/nat-POSTROUTEIN`. 因為 `nat` 相關的操作都會被 `conntrack` 進行快取幫忙做掉了。

我們將封包了來回搭配之前的圖表整理一下。

![Imgur](https://i.imgur.com/AuCxab9.png)v
因為容器與容器之間的傳輸，基本上都是在 `Linux Bridge` 底下進行傳輸，所以 `ping` 產生的 `ICMP Request` 以及 `ICMP Reply` 都會走相同的路線來傳輸。


## Localhost To Container
這次的情境更為簡單，本機上面直接透過 `ping` 這個應用程式去連結到容器內部，這邊會直接使用容器的 `IP` 地址來進行溝通。

情境圖如下，相對於上述的 `Container To Container`, 這次的封包不是完全的 `Layer2` 轉發就可以處理的，會牽扯到本機上面的 `Ping` 程式，這意味者 `Layer3` 的部分也會出現。
![Imgur](https://i.imgur.com/v8xwHDe.png)


### Setup ebtables
基本上 `ebtables` 的指令與前述相同，沒有部份需要修改，可以繼續使用前述的 `ebtables` 指令。

```bash=
#!/bin/bash

insert() {
    ebtables -t broute -I BROUTING --log --log-prefix 'ctc/ebtable/broute-BROUTING' --log-level debug
    ebtables -t nat -I PREROUTING --log --log-prefix 'ctc/ebtable/nat-PREROUTE' --log-level debug
    ebtables -t nat -I POSTROUTING --log --log-prefix 'ctc/ebtable/nat-POSTROUTE' --log-level debug
    ebtables -t filter -I INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
    ebtables -t filter -I OUTPUT --log --log-prefix 'ctc/ebtable/filter-output' --log-level debug
    ebtables -t filter -I FORWARD --log --log-prefix 'ctc/ebtable/filter-forward' --log-level debug
}

delete() {
    ebtables -t broute -D BROUTING --log --log-prefix 'ctc/ebtable/broute-BROUTING' --log-level debug
    ebtables -t nat -D PREROUTING --log --log-prefix 'ctc/ebtable/nat-PREROUTE' --log-level debug
    ebtables -t nat -D POSTROUTING --log --log-prefix 'ctc/ebtable/nat-POSTROUTE' --log-level debug
    ebtables -t filter -D INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
    ebtables -t filter -D OUTPUT --log --log-prefix 'ctc/ebtable/filter-output' --log-level debug
    ebtables -t filter -D FORWARD --log --log-prefix 'ctc/ebtable/filter-forward' --log-level debug
}

check() {
    count=`ebtables-save | grep ctc| wc -l`
    if [ "$count" == "0" ]; then
        echo "Delete Success"
    else
        echo "Delete Fail, Use the ebtables-save to check what rules still exist"
    fi
}


if [ "$1" == "d" ]; then
delete
check
else
insert
fi
```

### Setup iptables
於 `iptables` 方面，我們新增了一條 `mangle OUTPUT` 來協助觀察封包的轉送，主要原因是本機的 `ping` 應用程式送出 `ICMP Request` 會牽扯到 `OUTPUT Chain`。

```bash=
#!/bin/bash

insert() {
    iptables -t mangle -I OUTPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-OUTPUT' --log-level debug
    iptables -t mangle -I PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t mangle -I POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -I PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-PREROUTE' --log-level debug
    iptables -t nat -I POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-POSTROUTE' --log-level debug
    iptables -t filter -I INPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-input' --log-level debug
    iptables -t filter -I OUTPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-output' --log-level debug
    iptables -t filter -I FORWARD -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-forward' --log-level debug
}

delete() {
    iptables -t mangle -D OUTPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-OUTPUT' --log-level debug
    iptables -t mangle -D PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t mangle -D POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -D PREROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-PREROUTE' --log-level debug
    iptables -t nat -D POSTROUTING -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/nat-POSTROUTE' --log-level debug
    iptables -t filter -D INPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-input' --log-level debug
    iptables -t filter -D OUTPUT -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-output' --log-level debug
    iptables -t filter -D FORWARD -s 172.18.0.0/16 -d 172.18.0.0/16 -j LOG --log-prefix 'ctc/iptable/filter-forward' --log-level debug
}

check() {
    count=`iptables-save | grep ctc| wc -l`
    if [ "$count" == "0" ]; then
        echo "Delete Success"
    else
        echo "Delete Fail, Use the iptables-save to check what rules still exist"
    fi
}


if [ "$1" == "d" ]; then
delete
check
else
insert
fi
```
### Test
當上述的規則都準備完畢之後，我們就可以開始來進行測試了。
由於這次是使用本機上面的 `ping` 指令來傳輸封包，所以測試的指令更為簡單

接者執行下列指令，請先確保(172.18.0.2)是 容器 `nginx` 的 `IP` 地址
```bash=
ping 172.18.0.2 -c1
```

接下來馬上使用 `sudo dmesg -ct` 來觀察結果，然後我將結果分成 `ICMP Request` 以及 `ICMP Reply` 兩個部分來觀察。

```bash=
ctc/iptable/mangle-OUTPUTIN= OUT=docker0 SRC=172.18.0.1 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=15859 DF PROTO=ICMP TYPE=8 CODE=0 ID=30580 SEQ=1
ctc/iptable/filter-outputIN= OUT=docker0 SRC=172.18.0.1 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=15859 DF PROTO=ICMP TYPE=8 CODE=0 ID=30580 SEQ=1
ctc/iptable/mangle-POSTROUTEIN= OUT=docker0 SRC=172.18.0.1 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=15859 DF PROTO=ICMP TYPE=8 CODE=0 ID=30580 SEQ=1
ctc/iptable/nat-POSTROUTEIN= OUT=docker0 SRC=172.18.0.1 DST=172.18.0.2 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=15859 DF PROTO=ICMP TYPE=8 CODE=0 ID=30580 SEQ=1
ctc/ebtable/filter-output IN= OUT=veth5ead5c7 MAC source = 02:42:db:a1:f2:79 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
ctc/ebtable/nat-POSTROUTE IN= OUT=veth5ead5c7 MAC source = 02:42:db:a1:f2:79 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
```

1. 由於封包是直接從本機的 `Ping` 出發的，所以會先從 `Layer3` 開始傳送封包，因此第一個遇到的就會是 `iptables` 相關的規則
2. 這邊可以觀察到因為封包是從本機出去的，所以其實 `IN=` 的欄位一直都是空的，因為其實也不重要。
3. 因為目標容器是在 `Linux Bridge` 底下，所以封包會先查到 `docker0`, 最後依賴 `Layer2` 去轉發送出去。

```bash=
ctc/ebtable/broute-BROUTING IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:db:a1:f2:79 proto = 0x0800
ctc/ebtable/nat-PREROUTE IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:db:a1:f2:79 proto = 0x0800
ctc/iptable/mangle-PREROUTEIN=docker0 OUT= PHYSIN=veth5ead5c7 MAC=02:42:db:a1:f2:79:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.18.0.1 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=55481 PROTO=ICMP TYPE=0 CODE=0 ID=30580 SEQ=1
ctc/ebtable/filter-input IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:db:a1:f2:79 proto = 0x0800
ctc/iptable/filter-inputIN=docker0 OUT= PHYSIN=veth5ead5c7 MAC=02:42:db:a1:f2:79:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.18.0.1 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=55481 PROTO=ICMP TYPE=0 CODE=0 ID=30580 SEQ=1
```

1. `ICMP Reply` 的方向是從容器回到本機的 `Ping` 應用程式，因此進入點就是`Linux Bridge`, 這意味者一定是從 `ebtables/broute-BROUTING`  開始
2. 查詢完相關的 `Bridging Table` 以及 `Routing Table`, 最後決定要將封包送到 `Ping` 的應用程式，因此會走到 `INPUT Chain` 這邊來處理。

最後我們將上述的流向給合併起來觀看，在這個範例之中因為 `ICMP Request` 以及 `ICMP Reply` 是不同的走向。所以在下圖中。我們紫色的代表是 `ICMP Request` 的走向，而藍色代表的是 `ICMP Reply` 的走向。

![Imgur](https://i.imgur.com/6CDXA6w.png)

## Wan To Container
終於到了最後一個情境，這個情境也是最多人常用的情境。我們的容器本身再創立的時候，會透過 `-p 5566:80` 的方式將本機的 `5566` 連接埠串通到容器內的 `80` 連接埠.

接者外部的網路透過 `5566` 連結埠來存取對應的容器內容。

因此在這個範例中，我們打算從外部網路透過 `5566` 連結埠來存取是先建立好的 `Nginx` 容器。

接者透過 `iptables/ebtables` 的記錄來觀察在這種情境下，封包會怎麼傳輸。
此外，由於我們還有透過 `5566` 連結埠轉換到 `80`連結埠的需求，所以在我們觀察的 `iptables/ebtables` 的結果中，應該也要可以看到封包資訊的變換(IP/Port/MAC Address)
![Imgur](https://i.imgur.com/Ongf166.png)

### Setup ebtables
在 `ebtables` 方面，規則基本上沒有太多變化，繼續依照之前的用法即可。

```bash=
#!/bin/bash

insert() {
    ebtables -t broute -I BROUTING --log --log-prefix 'ctc/ebtable/broute-BROUTING' --log-level debug
    ebtables -t nat -I PREROUTING --log --log-prefix 'ctc/ebtable/nat-PREROUTE' --log-level debug
    ebtables -t nat -I POSTROUTING --log --log-prefix 'ctc/ebtable/nat-POSTROUTE' --log-level debug
    ebtables -t filter -I INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
    ebtables -t filter -I OUTPUT --log --log-prefix 'ctc/ebtable/filter-output' --log-level debug
    ebtables -t filter -I FORWARD --log --log-prefix 'ctc/ebtable/filter-forward' --log-level debug
}

delete() {
    ebtables -t broute -D BROUTING --log --log-prefix 'ctc/ebtable/broute-BROUTING' --log-level debug
    ebtables -t nat -D PREROUTING --log --log-prefix 'ctc/ebtable/nat-PREROUTE' --log-level debug
    ebtables -t nat -D POSTROUTING --log --log-prefix 'ctc/ebtable/nat-POSTROUTE' --log-level debug
    ebtables -t filter -D INPUT --log --log-prefix 'ctc/ebtable/filter-input' --log-level debug
    ebtables -t filter -D OUTPUT --log --log-prefix 'ctc/ebtable/filter-output' --log-level debug
    ebtables -t filter -D FORWARD --log --log-prefix 'ctc/ebtable/filter-forward' --log-level debug
}

check() {
    count=`ebtables-save | grep ctc| wc -l`
    if [ "$count" == "0" ]; then
        echo "Delete Success"
    else
        echo "Delete Fail, Use the ebtables-save to check what rules still exist"
    fi
}


if [ "$1" == "d" ]; then
delete
check
else
insert
fi
```


### Setup iptables
不同於 `ebtables`，在 `iptables` 這邊的修改比較多，原因如下
1. 此情境屬於 `Wan To Container`, 這意味牽扯到不同網段的傳輸
2. 因為我操作的環境算是很乾淨，所以我針對 `Wan IP` 以及 `Container IP` 來作為封包的條件
3. 在我的環境中，我的 `Wan` 是 `172.17.0.1` 而 `Container` 是 `172.18.0.2`. 所以我規則會針對 `172.17.0.0/16` 以及 `172.18.0.0/16` 來設定。

```bash=
#!/bin/bash

insert() {
    iptables -t mangle -I PREROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t nat -I PREROUTING -p tcp  -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-PREROUTE' --log-level debug
    iptables -t mangle -I PREROUTING -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t nat -I PREROUTING -p tcp  -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-PREROUTE' --log-level debug

    iptables -t filter -I FORWARD -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/filter-forward' --log-level debug
    iptables -t filter -I FORWARD -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/filter-forward' --log-level debug
    iptables -t mangle -I FORWARD -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-FORWARD' --log-level debug
    iptables -t mangle -I FORWARD -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-FORWARD' --log-level debug

    iptables -t mangle -I POSTROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t mangle -I POSTROUTING -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -I POSTROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-POSTROUTE' --log-level debug
    iptables -t nat -I POSTROUTING -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-POSTROUTE' --log-level debug
 }

delete() {
    iptables -t mangle -D PREROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t nat -D PREROUTING -p tcp  -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-PREROUTE' --log-level debug
    iptables -t mangle -D PREROUTING -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-PREROUTE' --log-level debug
    iptables -t nat -D PREROUTING -p tcp  -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-PREROUTE' --log-level debug

    iptables -t filter -D FORWARD -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/filter-forward' --log-level debug
    iptables -t filter -D FORWARD -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/filter-forward' --log-level debug
    iptables -t mangle -D FORWARD -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-FORWARD' --log-level debug
    iptables -t mangle -D FORWARD -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-FORWARD' --log-level debug

    iptables -t mangle -D POSTROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t mangle -D POSTROUTING -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/mangle-POSTROUTE' --log-level debug
    iptables -t nat -D POSTROUTING -p tcp -d 172.18.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-POSTROUTE' --log-level debug
    iptables -t nat -D POSTROUTING -p tcp -d 172.17.0.0/16 -j LOG --log-prefix 'wtc/iptable/nat-POSTROUTE' --log-level debug
}

check() {
    count=`iptables-save | grep wtc| wc -l`
    if [ "$count" == "0" ]; then
        echo "Delete Success"
    else
        echo "Delete Fail, Use the iptables-save to check what rules still exist"
    fi
}


if [ "$1" == "d" ]; then
delete
check
else
insert
fi
```


### Test
在測試方面，我一開始本來是採用 `curl` 的方式去連線 `nginx` 容器，但是其實 `curl` 做了太多事情了，除了一開始的 `TCP` 三方交握連線外，還包含了 `HTTP GET`。 對於我們只想要單純觀察 `Wan To Controller` 這來回的連線來說，這其實做了太多事情了。

為了簡化整個觀察結果，我最後決定採用 `telnet` 的方式，單純建立 `TCP` 連線就好。而整個 `TCP` 的三方交握連線其實是三個封包的傳輸，所在觀察的結果中可以觀察到三個部分的連線。

然而在我們的觀察目標中，我們只需要觀察前兩個連線就好，畢竟這樣已經足夠讓我們去觀察 `Wan To Controller` 的傳輸過程中， `iptables/ebtables` 會如何影響我們的連線。

待一切規則都準備好後，在你外網的機器上，執行下列指令。
這邊要注意的是，我測試機器的對外`IP` 地址是 `172.18.8.211`，而我本身主機的`IP`地址是 `172.17.8.1`. 這邊請調整成自己的環境
```bash=
[18:13:20] hwchiu ➜ ~» telnet 172.17.8.211 5566
Trying 172.17.8.211...
Connected to 172.17.8.211.
Escape character is '^]'.
```

這時候馬上透過 `dmesg -ct` 去收集封包，可以得到類似下列的結果，我將結果整理，只收集 `TCP` 三方教握的前兩方傳輸就好
```bash=
ctc/iptable/mangle-PREROUTEIN=enp0s8 OUT= MAC=08:00:27:ff:b2:c4:0a:00:27:00:00:02:08:00 SRC=172.17.8.1 DST=172.17.8.211 LEN=64 TOS=0x00 PREC=0x00 TTL=64 ID=0 DF PROTO=TCP SPT=63584 DPT=5566 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
ctc/iptable/nat-PREROUTEIN=enp0s8 OUT= MAC=08:00:27:ff:b2:c4:0a:00:27:00:00:02:08:00 SRC=172.17.8.1 DST=172.17.8.211 LEN=64 TOS=0x00 PREC=0x00 TTL=64 ID=0 DF PROTO=TCP SPT=63584 DPT=5566 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
ctc/iptable/mangle-FORWARDIN=enp0s8 OUT=docker0 MAC=08:00:27:ff:b2:c4:0a:00:27:00:00:02:08:00 SRC=172.17.8.1 DST=172.18.0.2 LEN=64 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=63584 DPT=80 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
ctc/iptable/filter-forwardIN=enp0s8 OUT=docker0 MAC=08:00:27:ff:b2:c4:0a:00:27:00:00:02:08:00 SRC=172.17.8.1 DST=172.18.0.2 LEN=64 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=63584 DPT=80 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
ctc/iptable/mangle-POSTROUTEIN= OUT=docker0 SRC=172.17.8.1 DST=172.18.0.2 LEN=64 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=63584 DPT=80 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
ctc/iptable/nat-POSTROUTEIN= OUT=docker0 SRC=172.17.8.1 DST=172.18.0.2 LEN=64 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=63584 DPT=80 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
ctc/ebtable/filter-output IN= OUT=veth5ead5c7 MAC source = 02:42:db:a1:f2:79 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
ctc/ebtable/nat-POSTROUTE IN= OUT=veth5ead5c7 MAC source = 02:42:db:a1:f2:79 MAC dest = 02:42:ac:12:00:02 proto = 0x0800
```

如同前述一樣，我這邊也針對一些有趣的封包內容進行討論
1. 因為我們是透過 `docker run -p 5566:80`, 所以我們傳輸的 `5566` 連接埠會被轉換成 `80` 連接埠。 可以觀察到第三個規則 `mangle-FORWARD` 之後，`DPT=5566` 都變成了 `DPT=80`. 主要是因為經過了 `nat-PREROUTING` 後就被更動了。
2. 同上面的理由，可以觀察到封包的目標`IP`地址從原本的`DST=172.17.8.211` 被轉換成容器的`DST=172.18.0.2`.
3. 因為是從`Wan To Controller`, 所以封包會先從 `iptalbes` 開始跑，最後跑道 `Linux Bridge` 後才會進入到 `ebtables`
3. 最一開始封包的 `MAC Address` 的發送是 `0a:00:27:00:00:02 ->08:00:27:ff:b2:c4`. 但是一但到了 `ebtables` 那層，也就是經過 `docker0` 之後，你可以觀察到這時候的 `MAC address` 的流向變成 `02:42:db:a1:f2:79  -> 02:42:ac:12:00:02`. 這邊原理實際上跟 `IP` 封包傳輸有關，這邊不多敘述。

```bash=
ctc/ebtable/broute-BROUTING IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:db:a1:f2:79 proto = 0x0800
ctc/ebtable/nat-PREROUTE IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:db:a1:f2:79 proto = 0x0800
ctc/iptable/mangle-PREROUTEIN=docker0 OUT= PHYSIN=veth5ead5c7 MAC=02:42:db:a1:f2:79:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.17.8.1 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=0 DF PROTO=TCP SPT=80 DPT=63584 WINDOW=28960 RES=0x00 ECE ACK SYN URGP=0
ctc/ebtable/filter-input IN=veth5ead5c7 OUT= MAC source = 02:42:ac:12:00:02 MAC dest = 02:42:db:a1:f2:79 proto = 0x0800
ctc/iptable/mangle-FORWARDIN=docker0 OUT=enp0s8 PHYSIN=veth5ead5c7 MAC=02:42:db:a1:f2:79:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.17.8.1 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=80 DPT=63584 WINDOW=28960 RES=0x00 ECE ACK SYN URGP=0
ctc/iptable/filter-forwardIN=docker0 OUT=enp0s8 PHYSIN=veth5ead5c7 MAC=02:42:db:a1:f2:79:02:42:ac:12:00:02:08:00 SRC=172.18.0.2 DST=172.17.8.1 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=80 DPT=63584 WINDOW=28960 RES=0x00 ECE ACK SYN URGP=0
ctc/iptable/mangle-POSTROUTEIN= OUT=enp0s8 PHYSIN=veth5ead5c7 SRC=172.18.0.2 DST=172.17.8.1 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=80 DPT=63584 WINDOW=28960 RES=0x00 ECE ACK SYN URGP=0
```

這邊是封包的回程，是由 `Container To Wan` 的方向
1. 封包是從 `ebtables` 開始，因為是從 `Linux Bridge` 底下的網卡收到容器傳出來的封包，接下來透過各種轉發，最後從 `iptabes` 那邊轉發出去到外部網路。
2. 觀察最後一個 `mangle-postrouting` 可以看到 `IP/Port/Mac` 都還沒有被轉換，這些都會在 `nat-postrouting` 這邊去處理，但是因為 `conntrack` 的關係，這些操作會在 `kernel` 給快取執行掉了。若要真的觀察可以透過 `tcpdump` 的方式去監聽封包。


在看到了這兩種的情境後，我們將彼此的流向圖給整理一下，如下圖。
圖中藍色的連線則是 `Wan To Container` 而紫色則是 `Container To Wan` 的封包流向。

基本上因為牽扯到 `Layer3/Layer2` 的處理，封包都會經過 `iptables` 的 `filter Table/FORWARD Chain`. 如果有要針對防火牆去處理的話，可以在這邊去執行。

![Imgur](https://i.imgur.com/nzd69Ob.png)

## Summary
在本文中，我們嘗試透過 `iptables/ebtables` 本身的 `log` 模組來協助我們釐清於不同的拓墣情境中，封包之間的轉送會怎麼經過 `iptables/ebtables`。
總共有三種拓墣環境，分別是
1. Container To Container
2. Localhost to Container
3. Wan To Container

針對每個環境，我們都觀察封包的來回兩種狀態，除了觀察 `iptables/ebtables` 的走向之外，也順便觀察封包內容的變化。
只有真正的瞭解整個封包的傳輸行為，以及對應低 `iptables/ebtables` 的走向，未來在管理 `iptables/ebtables` 才能夠更精準的去設定相關的規則來滿足自己的需求。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/D7RZGWrNK

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