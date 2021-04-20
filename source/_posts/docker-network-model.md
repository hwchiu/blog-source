---
title: Docker Network - 網路模型
keywords: 'Docker,Network,CNI,Kubernetes'
tags:
  - Docker
  - Network
  - Kubernetes
description: >-
  本篇文章從入門的概念來介紹 Docker 的網路模型，透過對其使用上的瞭解，可以幫助我們去理解容器之間網路的使用，對於未來學習 Kubernetes
  時會得心應手
abbrlink: 34885
date: 2020-10-26 00:14:22
---

Docker Network - 網路模型
========================

# 前言

隨者 Kubernetes 近年風潮崛起， 容器（Container）的概念顯然已變成一個標配，特別是針對管理平台相關的職缺，過去職缺對於面試者的要求，可能是聽過或摸過 Docker，而近年則是 Docker/Container 為主要需求，直到如今，職缺對於 Kubernetes 的要求則取代了過往 Docker 的地位。根據我的觀察，在矽谷我所看到的工作職缺可謂滿滿的 Kubernetes。

Kubernetes 作為一個容器管理平台，其架構非常複雜，相對於過往單節點 Docker 的使用情境來說，學習門檻高了不少，這部份包含了容器使用，儲存空間使用以及網路使用。

儘管如此， Kubernetes 與 Docker 還是有很多相似之處，掌握好 Docker 的實作細節對於學習與理解 Kubernetes 還是有十分多的幫助，特別是在容器網路（Container Networking）這一塊很多基本細節是完全一致的。

> 舉例來說，Kubernetes CNI 內常常使用的 bridge CNI 其實運作概念跟 `docker run --network=bridge` 是完全一樣的。

因此我在本篇文章中，將會為大家帶來 Docker 的幾種網路模型的介紹。

相關程式碼都可以於這邊找到 [technologynoteniu/bloglab-source code](https://github.com/technologynoteniu/bloglab/tree/main/docker_network_basic_1)

# 使用

使用 `docker` 指令創建容器時，我們可以透過 `--network` 該參數來指定想要創建的網路模型，這邊會針對下列幾種介紹其概念，架構以及用途。

1. None
2. Host
3. Bridge
4. Contaienr:$ID


## None

`None` 這個參數的意思就是告訴 Docker Engine 不要幫我管理任何任何網路功能，只要建立一個隔離網路空間（Network namespace）就好。

### 範例指令

```bash
$ docker run --network=none -d --name none hwchiu/netutils
$ docker exec none ifconfig -a
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

在我們的範例當中可以看到，預設就只有一個 lo(loopback) 的介面，沒有其他任何網路卡介面，所以這個 container 也沒有對外上網的能力。

### 網路環境觀察

這邊用兩張不同視角來看待這個行為，由於本篇文章都會採用相同的概念去解讀不同網路模型，因此這邊詳細介紹兩種視角的概念。

**圖左**：這邊想要介紹的是以系統底層的角度去觀察網路，中間的灰色線將其分為上半部分的 UserSpace, 以及下半部分的 Kernel Space。
本範例中下圖會有不同顏色變化，代表不同的網路空間，每個網路空間彼此網路隔離。

**圖右**：這邊提供一個比較簡略的介紹，主要會從使用者的角度去觀察，由圖例來說明網路元件的關係上有什麼變化。

**在瞭解上述概念後**，我們再來看一下如何去理解這張圖片：

**圖左**： 當創建一個全新的 Container 後，系統會幫我們在 Kernel 內創建一個全新的網路空間（黃色區塊）來達到網路隔離的效果。不過因為我們沒有對這個隔離環境做任何設定，所以這個網路空間中只會有一個 **lo** 的網卡。

至於淺藍色的部份則代表著系統原先的網路空間，在此假設已經存在預設網卡 **eth0**。

**圖右**: 系統中產生了一個全新的 Container, 但是這個 Container 跟原生主機沒有任何互動，網路關係上就是 `毫無關係`。

![The different aspect of network namespace](https://i.imgur.com/jOnC3Qw.jpg)

### 使用情境

想要開發網路模型，或想要研究 Docker 網路，以及想要開發 CNI 的人都適合用這種模式創建乾淨網路，然後開始透過各種方式讓其能夠上網。

## Host

`Host` 這個參數的意思就是告訴 `Docker`，請不要幫我創造 **network namespace**，我不需要網路隔離，和宿主機共用相同的網路模型即可。

> 共用內容包含了 網卡，路由表，防火牆 ... 等

### 範例指令

```
$ docker run --network=host -d --name host hwchiu/netutils
$ docker exec -it host ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 0e:e5:e9:25:d8:41 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:71:98:4c:2a brd ff:ff:ff:ff:ff:ff

$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 0e:e5:e9:25:d8:41 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:71:98:4c:2a brd ff:ff:ff:ff:ff:ff
```

透過上述指令的創造，可以發現該 container 內所看到的網卡資訊與外部主機的內容是完全一樣的，除了使用 **ip link**外，其他的指令如 **ip addr**, **ifconfig**, **iptables-save**, **ipvsadm** 等都會看到相同內容。

> network port 都會共用，因此如果外面已經有服務使用 port 80, 你就不能再跑一個 port 80 於相同的 address 上。

### 網路環境觀察

**圖左**： 當創建一個全新的 Container 後，我們不需要任何網路隔離功能，因此 Kernel 內並沒有任何新的網路空間被創造。我們的 Container 會直接使用預設網路空間內的所有網路資源，譬如網卡 **eth0**。

**圖右**: 系統中產生了一個全新的 Container，這個 Container 跟原生主機共用網路空間，因此 Container 看到的網路環境會與宿主機是完全一致。

![](https://i.imgur.com/H3pcaev.jpg)


### 使用情境

1. 想要直接存取 Container，不希望封包會被其他路徑處理
    a. 網路效能與存取本機相同
2. 特別注意 Port Number 的使用，若啟用相同服務時就很容易發生衝突
3. 容器需要使用特殊硬體資源，但掛載到容器導致相對麻煩時，我們則會使用這種方式共用宿主機的資源
4. 容器本身會需要對宿主機的網路環境進行操作或監控時


## Bridge

`Bridge` 這個參數的意思就是告訴 `Docker` 請幫我創造全新的 **network namespace**，然後我想要透過 **Linux Bridge** 來與原生網路有互動的能力

> 這部份先忽略 iptables 的任何規則

這個模型也是 Docker 中預設的網路模型，會幫你執行下列步驟：

1. 你於容器內創建一張網卡，並且指派相關的 IP addresses
2. 於主機創建一個 **Linux Bridge**
3. 透過 **veth** 幫你把 **container** 與 主機 這兩個不同的網路空間給串連一起

> veth 概念複雜，在此暫時不探討太多，只要知道是一個特殊的方式來串連不同網路空間。


### 範例指令


下述的指令比較複雜，首先我們會先創建一個使用 **bridge** 模型的 container。

然後我們可以透過 **ip addr** 觀察到裡面有兩張網卡，分別是 **lo** 以及 **eth0**， 這邊要注意的是 **eth0** 後面有一個數字 **183**，這個數字跟 **veth** 的概念有關，不過請容我在此先略過。

接者我們到主機方面去呼叫 **ip link** 然後用 **183** 來搜尋，會發現系統上有一個網路介面跟 **183** 有關係。

最後我們透過 **brctl** 工具可以發現系統上有一個 **Linux Bridge** 叫做 **docker0**，而剛剛發現的網卡正是被綁定在這個 **docker0** 當中。


```
$ docker run --network=bridge -d --name bridge hwchiu/netutils

$ docker exec bridge ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
182: eth0@if183: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
$ ip link | grep 183
183: veth980af09@if182: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default

$ brctl show | grep veth980af09
docker0         8000.024271984c2a       no              veth980af09
```

### 網路環境觀察

**圖左**: 這個架構比較複雜，分成幾個角度來看：

1. 容器創造後，會於自己的網路空間（黃色區塊）內額外創造一張新的網卡 **eth0**，因此，網路空間內就會有兩張網卡，分別是 **lo** 以及 **eth0**
2. 外部主機的預設網路空間（水藍色區塊）中，會創建一個全新的 **Linux Bridge**：**docker0**，並且透過 **veth** 的機制與 Container 內的 eth0 串接起來

> veth 概念複雜，在此暫時不探討太多，只要知道是一個特殊的方式來串連不同網路空間。

**圖右**：系統中除了產生 Container 之外，還會產生一個全新的 Linux Bridge，並且透過 **veth** 的方式將 **Linux Bridge** 與 Container 串連一起。

![](https://i.imgur.com/VKnduEq.jpg)

如果這時候我們再額外建立一個新的 Container（粉色區塊），那結果會如下圖

1. 全新的 Contaienr 會創造一個新的網路空間（綠色區塊），並且設定好網路介面。
2. 因為當前系統上已經有 **docker0** 可以使用，因此 Docker 不會創建新的 Bridge 來橋接 Container
3. 透過 **veth** 將新的網路空間（綠色區塊）與外部主機的網路空間（水藍色區塊）串連起來
4. 從右邊圖片來看，系統上有愈來愈多的 **veth** 網卡，這些網卡其實都連接到不同的 Container

![](https://i.imgur.com/skON4A6.jpg)

### 使用情境

- Docker 預設網路模型，能夠對外上網
> 需要與 iptables 合作來達到存取外網，這篇文章先跳過 iptables 部分
- Container 之間可以透過 IP 的方式互通
- 對網路沒有要求，單節點彼此能通就好

## Container

`Container:$ID` 這個參數的意思就是告訴 `Docker` 不要幫我創造新的網路空間，取而代之，使用現有的 Container 的網路空間，和它共處於相同的網路環境中。因此，這兩個 Container 將會看到一樣的網路介面、路由表 ... 等網路相關資訊。

### 範例指令

```
$ docker run --network=container:$(docker ps --filter name=bridge -q) -d --name co_container hwchiu/netutils

$ docker exec co_container ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1366 (1.3 KB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

$ docker exec bridge ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1366 (1.3 KB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

可以看到創建後，這兩個容器內所看到的網卡內容都會一模一樣。

### 網路環境觀察

**圖左**: 這個架構比較複雜，分成幾個角度來看：

1. 原本的容器（淺藍色）以 Bridge 模式創立，因此系統會產生 **bridge** 環境所使用到的所有資源
2. 接著新的容器（粉紅色）以共享網路的方式創立，新容器要掛載到藍色容器的網路空間中，所以不會有新的網路空間，會共享藍色容器的網路空間（黃色區塊），包含 **eth0**, **lo** 等

**圖右**: 系統中產生 Container 後，會直接與目標 Container 共享網路空間，因此會看到這兩個 Container 共享同一張網卡 eth0。

> 如果本來的 Container 是 None, 那這種情況就是兩個人一起 None，簡而言之就是共享網路環境而已。

![](https://i.imgur.com/n3XHucC.jpg)

### 使用情境

- 相同網路空間內的容器因為共享 **lo**，所以可以使用 localhost 來存取彼此服務
- 不同容器有存取需求，可以透過此方式享受到更快的網路存取速度
- Kubernetes 的 Pods 基於這個模型去實作，所以 Kubernetes Pod 裡面可以有多個 Containers 且彼此可以使用 localhost 來互相存取彼此的服務。


# 結論

本篇文章跟大家介紹了 Docker 的基本模型，並沒有涉及太多底層的技術細節，從不同的網路環境來認識這些模型的差異，同時我們也比較每種環境的使用時機，讓各位對於這些網路模型有更多認識。

這些虛擬網路環境其實都和我們時刻相處，瞭解這些網路架構的不同也會有助於思考整套系統的架構。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
