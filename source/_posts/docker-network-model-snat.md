---
title: Docker 網路入門篇(三) - 網路存取分析
keywords: 'Docker,Network,CNI,Kubernetes,Bridge'
tags:
  - Docker
  - Network
  - Kubernetes
description: 本篇文章探討 Docker Bridge 網路模型的運作過程，透過一系列步驟去拆解到底容器是如何對外上網
abbrlink: 51613
date: 2020-11-07 13:25:21
---

# 前言
本篇文章是 Docker 網路入門篇系列文第三篇，閱讀本文前要先有前面兩篇文章的基本概念，因此還不夠熟悉的讀者可以再次閱讀前面兩篇文章

[Docker Network - 網路模型](https://www.hwchiu.com/docker-network-model.html)
[Docker 網路入門篇(二) - Bridge 網路模型](https://www.hwchiu.com/docker-network-model-lab.html)

> 這系列的文章都會用比較使用者的角度來探討網路概念，比較不會去深度探討底層實作細節

# 本文

前篇文章中，我們透過指令的方式一步一步的打造一個 Bridge 網路模型，最後成功的讓兩個容器可以透過  ping 指令來互相存取，然而該環境中這兩個容器都沒有辦法對外上網，而且最後的過程中我們還留下了一個 iptables 指令的未解之謎，因此本篇文章要來將這些資訊給釐清作為一個完結篇

這篇文章會從三個面向，配上三個主軸來進行探討，分別是

1. 容器間如何互相存取
2. 容器如何主動存取外部服務
3. 外部網路如何主動存取容器

其中第三點的功能就是 **docker -p** 這功能的意思，因此範例中也會仔細介紹如何使用

每個面向裡面都由三個主軸來探討，分別是
1. 封包要送給誰
2. 誰來處理封包
3. 誰來過濾封包

用技術層面來講上面三個概念的話，大概是下列這些內容，不過本文不會太細究每個元件的內容
1. routing table + forwrding table
2. kernel + iptables + conntrack
3. iptables

# 環境

```
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.3 LTS
Release:        18.04
Codename:       bionic

$ uname -a
Linux k8s-dev 4.15.0-72-generic #81-Ubuntu SMP Tue Nov 26 12:20:02 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

$ docker --version
Docker version 19.03.13, build 4484c46d9d
```

實驗所有步驟都可以於 [GitHub Repo](https://github.com/technologynoteniu/bloglab) 中找到

# 容器間如何存取

本實驗是基於上次環境來進行後續觀察，到底一個簡單的 ICMP 封包可以通實際上系統中到底做了什麼事情。

可以直接執行範例 Repo 中的 **docker_network_basic_3/lab1.sh** 來打造上次的環境

首先，我們的環境中，特別設定了同網段的封包，所以這個環境中的存取就會相對簡單

整個環境架構圖如下
架構中兩個容器 C1 & C2 透過我們自行創造的 Linux Bridge(hwchiu0) 以及相關的 **veth** 虛擬網卡來串間彼此。

![](https://i.imgur.com/XhNnxAq.jpg)


## 封包要送給誰

之前我們透過 **ifconfig** 幫 **eth0** 設定 IP 後，Kernel 會針對該網卡設定一個 Routing 規則，告訴系統說，什麼樣的封包，往什麼樣的網卡送出去

我們來看一下當前兩個容器 C1 & C2 分別的狀態長怎樣
```bash=
○ → docker exec c1 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.55.66.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0

○ → docker exec c2 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.55.66.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0

```

這兩個訊息的概念都很簡單，就是告訴系統，當未來看到 **10.55.66.0/24** 的封包，就往 eth0 這張網卡送出去，但是因為 eth0 網卡是由 **veth** 組成的， **veth** 就如同水管一樣，從左邊進去，右邊就會出來，因此送到 **eth0** 的封包就會馬上從另一端的 **veth0/veth1** 跑出來。

概念如下圖
根據上述觀念，當前兩個容器往 **10.55.66.0/24** 的封包最後都會從宿主機身上的 **veth0/veth1** 上出現。

![](https://i.imgur.com/YxMBsYX.jpg)


## 誰來處理封包

當封包進入 eth0 網卡，之後從 veth0/veth1 出現後，因為 **veth0/veth1** 本身是掛在 Linux Bridge (hwchiu0) 身上，因此封包就會交由 **Linux Bridge** 來處理！

這邊的處理分成兩個部分(這邊不談實際封包處理順序)
1. Linux Bridge 內部會有一個機制用來決定如何轉送封包
2. ebtables (本文忽略這個概念)

Linux Bridge 的轉送機制是基於 **MAC Address** 來決定封包怎麼轉送，而這份轉換表可以稱為 Forwarding Table，根據 **MAC Address** 來轉發封包

下圖為一個範例，當封包進入到 Linux Bridge 後，其會根據封包的目標 **MAC Address** 是誰，來決定從哪個 **Port** 送出去。

因此我們的範例中，C1/C2 容器之間的封包就是透過這種機制來處理轉發。
![](https://i.imgur.com/Cns14pD.jpg)


## 誰來過濾封包

有了上述的規則後，C1/C2 容器之間要可以互相傳輸封包了，但是實務上卻發現會有問題，我們的 PING 不會通，原因是因為 **iptables** 偷偷介入來進行處理，並且將不符合規則的封包都丟棄。

這邊有兩個議題
1. iptables 為什麼偷偷介入來處理
2. 為什麼不符合規則的封包要丟棄，而不是符合規則的封包才丟棄

**iptables** 預設不會干擾任何 **Linux Bridge** 轉發的封包，這邊是有個開關要打開，也是 **docker** 安裝後會幫忙打開的開關

```bash=
→ cat /proc/sys/net/bridge/bridge-nf-call-iptables
1
```

這邊我們可以做個實驗，步驟是
1. 確認當前網路不通
2. 將上述開關關閉，讓 iptables 不要干涉
3. 重新發送 ping

這時候就會發現容器之間可以透過 ICMP 傳送封包了
```bash=
$ docker exec -it c1 ping 10.55.66.3 -c1 -W1
PING 10.55.66.3 (10.55.66.3) 56(84) bytes of data.

--- 10.55.66.3 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms


$ echo "0" | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables
0

$ docker exec -it c1 ping 10.55.66.3 -c1 -W1
PING 10.55.66.3 (10.55.66.3) 56(84) bytes of data.
64 bytes from 10.55.66.3: icmp_seq=1 ttl=64 time=0.025 ms

--- 10.55.66.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.025/0.025/0.025/0.000 ms

$ echo "1" | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables
1
```

當 **iptables** 被告知要干涉 **Linux Bridge** 的封包管理後，所有經過的封包都會讓 **iptables** 來進行檢查

Docker 的環境之中，採取的是白名單機制，若沒有告知要通過，則將該封包給丟棄，所以這也是為什麼我們的封包不會通過。
這邊有兩種解決方法
1. 修改成黑名單的概念，預設通過封包
2. 加入相關規則，讓我們的封包可以通過

前一篇文章的實驗就是針對 (1) 進行操作,這邊透過 **iptables -P FORWARD ACCEPT** 去說明，對於 **FORAWRD** 封包的處理，預設就是 **ACCEPT** 去接納他們

```bash=
$ docker exec -it c1 ping 10.55.66.3 -c1 -W1
PING 10.55.66.3 (10.55.66.3) 56(84) bytes of data.

--- 10.55.66.3 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms


$ sudo iptables -P FORWARD ACCEPT

$ docker exec -it c1 ping 10.55.66.3 -c1 -W1
PING 10.55.66.3 (10.55.66.3) 56(84) bytes of data.
64 bytes from 10.55.66.3: icmp_seq=1 ttl=64 time=0.039 ms

--- 10.55.66.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.039/0.039/0.039/0.000 ms

# recover
$ sudo iptables -P FORWARD DROP

```


## 結論

針對同節點上不同容器之間的傳輸，基本上都是依靠 **Linux Bridge** 來幫忙處理，藉由 **Forwarding Table** 來決定該怎麼傳輸封包，透過 **iptables** 來決定封包能不能通行。

Docker 預設的情況下會讓 **iptables** 介入 **Linux Bridge** 的處理，同時採由白名單機制，沒有符合規則的就一律丟棄，而 Docker 這邊的作法則是會加入相關的規則來打通封包連接。

![](https://i.imgur.com/b1veY6Z.jpg)

# 容器如何主動存取外部服務

上述討論了容器間的基本存取方式，有了上述的概念之後，我們接下來可以往下邁進去探討，如果容器想要存取外部服務，譬如 8.8.8.8 之類的外部網站時，到底該怎麼處理，這也是容器服務中最常使用的類型，畢竟沒有對外上網能力，很多事情都沒辦法完成。

這個範例中將探討如何讓一個容器作為我們的 Client 端，能夠透過 PING 這個指令來存取外部 8.8.8.8 服務器

架構圖如下方，最終目標是容器 C1 能夠用 ping 存取 8.8.8.8
![](https://i.imgur.com/PtgBuvH.jpg)


## 封包要送給誰
封包的轉送我們一開始都需要先決定，到底送給誰，這邊我們再次回顧一下當前容器 c1 內的路由表

```bash=
$ docker exec -it c1 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.55.66.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

這時候如果我們強行連接 8.8.8.8 的話，會得到相關錯誤,因為系統中根本不知道要怎麼轉送 8.8.8.8 的封包，完全沒有可以符合的規則使用
```bash=
$ docker exec -it c1 ping 8.8.8.8
connect: Network is unreachable
```
> 網路除錯時要格外小心與謹慎，我建議所有網路除錯都是先畫出架構圖，然後思考一下你認為封包該怎麼運作，對於所有關鍵點你有沒有辦法舉證他是對或是錯，藉由這個過程縮小可能出錯的範圍。

為了解決這個問題，我們可以 **明確告訴系統，看到 8.8.8.8 的封包該怎麼送**
但是這種思路造成的問題就是如果今天你想要存取 **1.1.1.1**，你就要額外的新規則，所以我們可以透過另外一種作法。

**沒有符合任何規則的話，就走預設走法吧！**
這也是系統實務上常見的作法，因為沒有人有辦法預料到你會想要連哪個網站，針對每個網站都寫一條規則實在是不太合理

有了這個想法後，我們下一個問題就是，那我要送給誰幫忙處理，這邊因為牽扯到 L3 路由的概念相對複雜，直接講結論就是
我們希望透過 **宿主機** 幫我們處理，如果宿主機本身就有能力存取外部網路，我們是不是能夠依賴宿主機幫我們處理，我們只要想辦法將封包送給宿主機，讓宿主機知道有封包要處理，請繼續往下弄

為了達成這個條件，我們需要進行下列設定
1. 給 Linux Bridge (hwchiu0) 一個 IP 地址
2. 告訴容器說，預設走法就是將封包送給 **Linux Bridge (hwchiu0)**

上述兩個概念轉換成系統指令如下
```bash=
$ sudo ifconfig hwchiu0 10.55.66.1 netmask 255.255.255.0
$ sudo docker exec -it c1 ip route add default via 10.55.66.1
$ sudo docker exec -it c1 ip route show
default via 10.55.66.1 dev eth0
10.55.66.0/24 dev eth0  proto kernel  scope link  src 10.55.66.2
```

設定完畢後，我們容器現在多了一個規則，預設情況下，就將封包透過 **eth0** 送出去，並且將其 閘道(Gateway) 設定成 **10.55.66.1**。
> 這邊不解釋 Gateway 的概念，想成我們想要透過 Linux Bridge 幫我們轉發，封包送給他就對了！

接下來針對系統上的 **eth0** 去監聽封包，看看這時候從 **container c1** 發送封包到 8.8.8.8 會怎麼樣

**為了避免 iptables 又來干擾過濾功能，我們先修改成預設封包都會通**

**開兩個視窗執行**

```bash=
$ sudo iptables -P FORWARD ACCEPT
$ sudo docker exec -it c1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
```

```bash=
$ sudo tcpdump -vvvnn -i eth0 icmp
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
19:01:20.425931 IP (tos 0x0, ttl 63, id 60779, offset 0, flags [DF], proto ICMP (1), length 84)
    10.55.66.2 > 8.8.8.8: ICMP echo request, id 16700, seq 1, length 64
19:01:21.445051 IP (tos 0x0, ttl 63, id 60899, offset 0, flags [DF], proto ICMP (1), length 84)
    10.55.66.2 > 8.8.8.8: ICMP echo request, id 16700, seq 2, length 64
```

透過上述指令可以觀察到，封包還是不通，沒有辦法得到正確的 ICMP 回應，但是我們可以從宿主機上面的 **eth0** 觀察到相關封包了，這些封包標示 **10.55.66.2** 想要送給 **8.8.8.8**。

只是目前都只有看到 **ICMP** 的請求，而沒有回應。

上述的所有流程我們用下列流程圖再次解釋

該圖片怎麼觀看
1. 上面白色框框代表不同元件，若元件上方有IP，則代表該元件的 IP 地址
2. 每個元件之間透過箭頭來描述封包流向，並且表明該流向中，封包內的 **IP** 地址是什麼，**由誰送給誰**
3. 封包流向下方描述的則是當前過程中，有哪些元件涉入進行處理

![](https://i.imgur.com/POi2cVr.jpg)


重新整理目前已知流程與思路

1. 當容器內發出一個送往 **8.8.8.8** 的封包，因為規則的關係，該封包會透過 **eth0** 送出去
2. 當封包從 **eth0** 送出去後，因為 **veth** 的特性，封包會到達系統上的 **veth0** 虛擬網卡
3. 從 **veth0** 進入到 **Linux Bridge** 的世界，這時候透過 **Forwarding Table** 的概念，最終該封包會進入到 Linux Bridge(hwchiu0) 本身。
4. hwchiu0 收到封包後，接下來都是 **kernel** 內的事情了，這邊太複雜，我們忽略
5. Kernel 最後透過本身的 **routing talbe** 去查詢，該怎麼轉送 **8.8.8.8** 的封包，然後根據規則送往宿主機上的 **eth0** 網卡
6. 封包送了出去，但是我們沒有辦法監聽到回來的封包

## 誰來處理封包
為什麼收不到封包，理由很簡單
我們送出去的封包來源是 **10.55.66.2**，大部分情況下我們這個網段都是 **private**，也就是私有網段，世界上可能有很多人都會使用 **10.55.66.2**，那這種情況下， **8.8.8.8** 根本不知道要怎麼把封包送回給 **10.55.66.2**

這時候我們要來思考，我們的宿主機是不是可以上網，是不是他的封包都回得來？
如果是的話，我們能不能叫宿主機幫忙好人作到底，把封包的來源也變成宿主機的 **IP** ? 然後宿主機再想辦法把封包轉送回給我們後面的容器即可

這個概念就是所謂的 Network Address Translation (NAT)，這個範例中我們想要修改封包的來源 **IP** 地址，因此這個行為我們會稱為 Source NAT (SNAT)。

為了達成這個目的，我們要透過 **iptables** 的規則來幫我們做，而 **iptables** 有多種用法可以滿足這個需求，我們決定採用最簡單的也是最常用的方式, **MASQUERADE**，這種動態 SNAT 的功能來處理

> 工商我的其他文章，從 Linux Kernel Source Code 來探討這個行為，不適合初學者看 [Linux NAT Masquerade 研究(上)
](https://www.hwchiu.com/iptables-masquerade.html)


我們使用下列規則，告訴 **iptables** 說，以後你只要看到 **10.55.66.2/32** 的封包，而且要從宿主機上面的 **eth0** 送出去的，請你順便幫忙修改封包來源，改成自己
```bash=
$ sudo iptables -t nat -I POSTROUTING -s 10.55.66.2/32 -o eth0 -j MASQUERADE
$ sudo docker exec -it c1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=18.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=15.0 ms
^C
```

這時候我們再透過 **tcpdump** 來監聽封包，就會觀察到一切都不同了!

```bash=
$ sudo tcpdump -vvvnn -i eth0 icmp
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
19:40:52.893673 IP (tos 0x0, ttl 63, id 39804, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.2.15 > 8.8.8.8: ICMP echo request, id 19069, seq 3, length 64
19:40:52.906130 IP (tos 0x0, ttl 62, id 11543, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.2.15: ICMP echo reply, id 19069, seq 3, length 64
19:40:54.962322 IP (tos 0x0, ttl 63, id 40179, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.2.15 > 8.8.8.8: ICMP echo request, id 19079, seq 1, length 64
19:40:54.977470 IP (tos 0x0, ttl 62, id 11559, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.2.15: ICMP echo reply, id 19079, seq 1, length 64
```

裡面的內容我們分成兩筆來看
```
19:40:52.893673 IP (tos 0x0, ttl 63, id 39804, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.2.15 > 8.8.8.8: ICMP echo request, id 19069, seq 3, length 64
```
我們可以看到出去的封包 **IP** 再也不是 **10.55.66.2** 了，而是宿主機本身的 **10.0.2.15**，你可能會想說 **10.0.2.15** 也是私有 IP，為什麼 **8.8.8.8** 本身可以回應，這是因為我的系統環境外面還有一層 **SNAT**，一個封包經歷過多次 SNAT 是滿正常且合理的，但是整個運作邏輯都是一致的

```bash=
[DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.2.15: ICMP echo reply, id 19069, seq 3, length 64
```
這個情況下我們也可以看到封包順利回來了，同時從容器中也可以看到 **ICMP** 有正常回應。

這邊補充一下，如果我們針對 **hwchiu0** 這張 Linux Bridge 的網卡去監聽封包，會得到下列資訊

```bash=
$ sudo tcpdump -vvvnn -i hwchiu0 icmp
tcpdump: listening on hwchiu0, link-type EN10MB (Ethernet), capture size 262144 bytes
19:47:55.128471 IP (tos 0x0, ttl 64, id 18861, offset 0, flags [DF], proto ICMP (1), length 84)
    10.55.66.2 > 8.8.8.8: ICMP echo request, id 19515, seq 1, length 64
19:47:55.146004 IP (tos 0x0, ttl 61, id 11872, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.55.66.2: ICMP echo reply, id 19515, seq 1, length 64
```

從 **hwchiu0** 的角度來看，他看到的封包都是 **10.55.66.2**，跟宿主機上面的 IP 沒有任何關係。

所以上述的概念我們用一樣的流程圖來看，這時候會變成怎麼樣

這份圖中，我們可以順利接收封包，所以多了回來封包的路線，其中 **IP** 特別用紅色底標示代表的是該封包是有被改過的。

![](https://i.imgur.com/tBBZIn7.jpg)

這邊重新整理所有思路，將其條列下來
1. 封包按照前面所有概念，一路送到 Linux Kernel
2. Linux Kernel 這時候查詢完 Routing Table 後，確認封包要送給 **eth0**
3. iptables 這時候會介入，透過 **MASQUERADE** 的功能來修改封包的來源 ip 地址
4. 封包的來源被修改成 **10.0.2.15**，最後 **8.8.8.8** 收到這些封包後一路送回來
5. 回應的封包到達 **eth0** 之後，**iptables** 再度介入，畢竟誰幫你轉換封包，誰就要幫你轉回來，幫你把封包的目標從 **10.0.2.15** 轉回最初的 **10.55.66.2**
> 其實更精準的說這邊還有 conntrack 的介入來處理，但是過於複雜，因此這邊忽略，我們懂大概念就好
6. 封包一路透過 Linux Bridge + Forwardnig Table + Veth 等元件回到容器手上

## 誰來過濾封包

到這邊為止，基本上我們的容器已經可以對外存取了，但是這邊有個前提是，我們事先修改 **iptables** 的規則，讓他預設是轉送封包，而不是丟棄


> Docker 安裝完畢後，會讓 iptables **FORWARD** 預設丟棄封包，所以 docker 會針對這一塊去撰寫規則讓你的容器可以對外上網


因此這個小節我們就來看看，要如何透過 iptables 來允許我們的封包通過

根據我們前面的流程加上最初的運作，其實 iptables 運作的地方有兩個
1. Linux Bridge 這邊會偷偷請 iptables 處理一次
2. Linux Kernel 往宿主機 eth0 這邊發送時也會有一次

這兩個環節於這個情境下，概念完全不同。
對於 Linux Bridge(hwchiu0) 來說，容器的封包會到達他身上，這邊是"到達"他身上，因此 **iptables** 內的規則就不是 **FORWARD** 這種轉發概念來處理，而是 **INPUT** 來處理。因此這邊我們就不需要特別處理
> INPUT chain 本身沒有被改成預設丟棄，因此都會通

相反的，第二個流程是 **LINUX KERNEL** 轉發封包，從 **eth0** 出發，因此這邊 **iptables** 的 **FORWARD** 規則表就要被考慮，所以我們要處理的就是這一段過程


首先我們將 **iptables** 中的 **FORWARD** 修改回預設丟棄封包，回歸到安裝好 **docker** 的設定。
```bash=
$ sudo iptables -P FORWARD DROP
$ sudo docker exec -it c1 ping 8.8.8.8
```

這時候你會發現封包不會通，透過 **tcpdump** 去監聽 **eth0** 的封包也沒有任何訊息，不過若是監聽 **hwchiu0** 則是會有封包


```bash=
$ sudo tcpdump -vvvnn -i eth0 icmp
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel

$ sudo tcpdump -vvvnn -i hwchiu0 icmp
tcpdump: listening on hwchiu0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:16:28.313865 IP (tos 0x0, ttl 64, id 60164, offset 0, flags [DF], proto ICMP (1), length 84)
    10.55.66.2 > 8.8.8.8: ICMP echo request, id 21202, seq 33, length 64
```

所以我們要透過 **iptables** 告訴系統，請允許我們的封包通過，這邊有非常多種思路
1. 針對來源與目的 IP 去處理
2. 針對來源與目的 網卡 去處理

用網卡會簡單很多，這部份沒有一定，完全看你設計， **docker** 會使用網卡來處理，這樣規則數量不會因為容器過多而變多。

以下是我們使用的規則，我們告訴 **iptables** 說
1. 從 hwchiu0 到 eth0 的封包，給他過！
2. 從 eth0 到 hwchiu0 的封包，給他過 !
```bash=
$ sudo iptables -t filter -I FORWARD -i hwchiu0 -o eth0 -j ACCEPT
$ sudo iptables -t filter -I FORWARD -i eth0 -o hwchiu0 -j ACCEPT
$ sudo docker exec -it c1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=16.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=15.8 ms
```


這時後來看架構圖，這個範例中， iptables 會有兩個地方干涉，分別是進入到 hwchiu0 以及宿主機 eth0 兩個地方。
但是兩個地方因為概念不同，使用的是 iptables 下不同的表，而我們這邊針對 FORWARD 表去處理，因為其預設是丟棄封包
![](https://i.imgur.com/ZyOxFzc.jpg)


## 總結

到這邊為止，我盡可能的用簡單的方式去描述，到底一個容器要對外上網中間會發生什麼事情，而這些事情平常我們都沒有感覺，是因為 **docker** 本身都幫我們處理完畢了，整體思路都完全一樣，不過對於 **iptables** 的規則會因為細度不同，所以下法也不太一樣。

# 總結
由於本篇文章已經過長，因此如何從外部存取容器就留到下篇文章再來分享

綜合本文與前文，這邊幫大家整理一下，基於 **Bridge** 網路模型下， Docker 容器是如何對外上網
1. 擁有 Linux Bridge，並且設定一個 IP
2. 創造容器，透過 veth 將容器與宿主機的 Linux Bridge 連接
3. 對容器內的網卡設定 IP，並且設定一個預設路由規則，讓 Linux Bridge 幫忙轉發對外封包
4. 設定相關 iptables 規則，讓系統轉發時，封包不會被丟棄
5. 設定 iptables SNAT 規則，讓我們往外的封包能夠有機會回來，最後輾轉回到容器手上

這一系列的規則看起來很多，但是其實整體都圍繞再 TCP/IP 的網路規則下，簡單的說就是
封包該怎麼走，誰幫忙處理封包，會不會有人攔截封包
將這三個思路整理下來，就可以很清晰的去分析封包的走向與除錯



# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=LG28Q5G

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
