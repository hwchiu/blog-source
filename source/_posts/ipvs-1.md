---
title: 'IPvS 學習手冊(一)'
keywords: 'Kubernetes,Network,Linux,Ubuntu, IPVS'
abbrlink: 58751
date: 2020-03-20 06:53:43
tags:
  - IPVS
  - Network
  - Linux
description: 本文作為 IPVS 系列文第一篇，主要跟大家粗略的介紹 IPVS 的概念以及相關用法，接下來會再仔細的探討一些更深層的概念，譬如實作細節以及一些使用技巧，最後再看看 Kubernetess 是如何與之互動的

---

# Preface

本篇文章作為系列文章的第一篇，該系列文希望能夠從概念到實作，從簡單到複雜來探討 IPVS (IP Virtual Server) 的概念
，目前規劃的主題包含
1. IPVS 的基本使用與概念
2. IPVS 與 Kubernetes 的整合
3. 從 Kernel 出發，看一下 IPVS 的底層實作

本文主要是從大方向出發，介紹 IPVS 的概念與用法，並且實際使透過 Docker 來建置一個測試環境。

# Introduction

如果你以前有透過 `NGINX` 或是相關的負載平衡器 (Load-Balancer) 來架設環境的話， **IPVS** 的功用對你絕對不陌生，非常類似。
**IPVS** 就是一個基於 **Layer 4** 的負載平衡器 (**Load-Balancer**), 支援 **TCP/UDP/STCP** 等協議來進行流量轉發，根據不同的演算法封包分配到後端不同的真實服務器。

可以使用下圖來簡單介紹一下其架構，圖片中我們總共有兩個元件
1. Director (IPVS Daemon)
2. Real Server

![](https://i.imgur.com/tb0iNdZ.png)

`Director` 代表的則是 **Load-Balancer** 也就是最前端接收封包處理的部份，對於 `Director` 來說，必須要有一個 **IP Address** 來給外部的應用程式連接用，之後我們都會稱其為 **VIP (Virtual IP)**.
而 **Real Server** 則代表後端提供服務的伺服器們。
這些伺服器可以跟 `Director` 屬於同一台機器之中，也可以屬於外部不同機器上的服務，只要 `Director` 能夠透過 **IPVS** 目前支援的幾種傳輸方法送到即可。

對於使用者來說，只需要連接到 **Director** 本身的 **VIP** 即可，並不需要知道後面的 **Real Server** 的 **IP** 地址，中間的轉送全部都由 **Director** 進行處理。
這意味所有的封包也都必須要交給 **Director** 進行處理，因此其本身的處理能力也就決定了這整個 **Load Balancing** 系統的處理能力與速度。


# Architecture

**IPVS** 的架構是由兩個部分組成，分別是 **UserSpace** 的控制介面以及 **Kernel Space** 的運算平面。

**User Space** 方面會透過 **ipvsadm** 這個工具來管理，可以用來創建所謂的 **Service**並且綁定不定數量的 **Real Server** 到該 **Service** 上。 本章節的最後面會有相關的示範

**Kernel Space** 方面則是有獨立的 **kernel module** (IPVS) 來進行相關功能的處理，這部分相當複雜。
該 **Kernel Module** 除了實現所謂的 **Load Balancer** 的功能外，本身其實也與 **Netfilter** 底層有掛勾，而這些掛鉤的行為都沒有辦法透過 **iptables**的指令來觀察到，所以其實本身的 **debug** 非常困難，基本上只能全權相信底層的運作，透過 **ipvsadm** 觀察到的都只有最上層的抽象概念 (Director 與 Real Servers) 的比對而已。

再次強調， **IPVS** 本身不會去看到封包應用層的資料，這意味並沒有辦法透過 HTTP Cookie, Session 等資訊作為判斷依據來轉發封包，整個處理流程都是基於 Layer 4 協定

# Usage

基本上所有的操作都依賴於 **ipvsadm** 這個工具，以下就來介紹幾個使用方法

## 創建 Virtual Service

```bash
$ ipvsadm -A virtual-service [-s scheduler] [-p [timeout]] [-M netmask] [--pe persistence_engine] [-b sched-flags]

virtual-service:
  --tcp-service|-t  service-address   service-address is host[:port]
  --udp-service|-u  service-address   service-address is host[:port]
  --sctp-service    service-address   service-address is host[:port]
  --fwmark-service|-f fwmark          fwmark is an integer greater than zero

```

**ipvsadm** 提供了眾多指令用來管理，其中 **-A(add)** 可以用來創建一組服務 **Virtual Service**, 大部分的 **Virtual Service** 都是由 **VIP:Port** + **Layer 4** 協定所組成，目前共支援 **tcp(-t),udp(-u),(sctp)** 這三種協定，此外由於跟 **Linux Kernel**綁定，所以其實也可以透過 **sk_buff**內的 **mark** 來進行處理。

```bash
$ sudo ipvsadm -A -t 172.17.8.101:80
```

上述指令會在系統中創建一個 **virtual service**, 其 **VIP**是 `172.17.8.101` 並且聽在 port `80` 上

接下來可以透過 `-a` 的方式針對已經存在的 **virtual service**來加入 **real servers**

```bash
k8s-dev:06:39:18 [~]root
$sudo ipvsadm -a -t 172.17.8.101:80 -r 172.18.0.2 -m
k8s-dev:06:39:24 [~]root
$sudo ipvsadm -a -t 172.17.8.101:80 -r 172.18.0.3 -m
```

其中 `-m` 的部分主要是跟 **director** 如何跟 **real server** 溝通，
這部分牽扯到不同的網路運作模式，本章節不進行太詳細探討，主要是先觀察 **service** 與 **real server** 的運作關係

接下來可以透過 **-l(-L)** 來觀察相關的資訊

```
$sudo ipvsadm -L
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.17.8.101:http wlc
  -> 172.18.0.2:http              Masq    1      0          0
  -> 172.18.0.3:http              Masq    1      0          0
```
上述可以看到幾個資訊
1. Service VIP (172.17.8.101), port =80 (HTTP port)
2. Real Servers
    - 172.18.0.2:80
    - 172.18,0.3:80
4. 目前沒有特別設定，所以上述兩個 Real Server 的權重一樣，這意味 `Director` 分配時會基於 50:50 去分配

# Experiment

接下來我們會透過下列的架構來進行實驗，該架構中我們會
1. 創立一個 **Service**
2. 透過 **Docker** 的方式建立多個 **Real Server** 並且加入到上述的 **Service** 中
3. 透過 **curl** 指令觀察結果
4. 嘗試改變 **Real Servers** 的權重，並且再次觀察結果


創建一個 `Service`, 該 IP **172.17.8.101** 真實存在於系統上
```bash=
ipvsadm -A -t 172.17.8.101:80
```

接下來透過 **Docker** 的方式創建兩個 **Real Server**，這邊採用 **Nginx** 並且配置不同的首頁內容

```bash=

sudo mkdir -p /nginx/A /nginx/B
echo "This is A" > /nginx/A/index.html
echo "This is B" > /nginx/B/index.html
docker run --rm -d -v "/nginx/A:/usr/share/nginx/html" --name nginx-A nginx
docker run --rm -d -v "/nginx/B:/usr/share/nginx/html" --name nginx-B nginx
```

接下來取得上述 **docker container** 的 IP 地址，並且加入到 **ipvs service** 中
```bash=
IP_A=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx-A)
IP_B=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx-B)

ipvsadm -a -t 172.17.8.101:80 -r $IP_A -m
ipvsadm -a -t 172.17.8.101:80 -r $IP_B -m
```

接下來我們可以透過 **curl** 的方式來透過 **IPVS**來存取背後的 **Nginx Docker Container**

```bash=
k8s-dev:06:00:35 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:00:39 [/home/vagrant]root
$curl 172.17.8.101
This is A
k8s-dev:06:00:39 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:00:39 [/home/vagrant]root
$curl 172.17.8.101
This is A
k8s-dev:06:00:39 [/home/vagrant]root
$curl 172.17.8.101
This is B
...
$ipvsadm -L -n --stats --rate
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  172.17.8.101:80                    13       85       45     5396     4617
  -> 172.18.0.2:80                       6       40       20     2548     2052
  -> 172.18.0.3:80                       7       45       25     2848     2565
```

預設情況下是 50:50 的權重分配，所以理論上兩者個 **conns** 比例應該要差不多
接下來嘗試改變看看權重

```bash=
k8s-dev:06:06:44 [/home/vagrant]root
$ipvsadm -e -t 172.17.8.101:80 -r $IP_B:80 -m -w 9
k8s-dev:06:09:50 [/home/vagrant]root
$ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.17.8.101:80 wlc
  -> 172.18.0.2:80                Masq    1      0          0
  -> 172.18.0.3:80                Masq    9      0          0
```

這時候權重變成 `1:9` 了，透過 **curl** 繼續瘋狂敲打看看
```bash=
k8s-dev:06:10:25 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:13 [/home/vagrant]root
$curl 172.17.8.101
This is A
k8s-dev:06:12:13 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:13 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:14 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:14 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:14 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:14 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:14 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:14 [/home/vagrant]root
$curl 172.17.8.101
This is B
k8s-dev:06:12:15 [/home/vagrant]root
$ipvsadm -L -n --stats --rate
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  172.17.8.101:80                    46      316      210    20180    21546
  -> 172.18.0.2:80                      10       68       40     4340     4104
  -> 172.18.0.3:80                      36      248      170    15840    17442
```

這時候再次觀察比例，會發現 **B** 明顯比 **A** 還要更多

# Summary

本文透過一個非常簡單也非常簡略的方式去介紹 **IPVS** 的用途以及用法，實際上背後還有很多原理需要理解，包含 **Director** 如何跟 **Real Server** 溝通，不同的 **Load Balancing**演算法, 底層如何實現以及要如何跟 **Kubernetes** 整合。

接下來的文章會嘗試一一解答上述的問題


## Reference
- http://www.linuxvirtualserver.org/Joseph.Mack/linuxexpo99/linuxexpo2.html
