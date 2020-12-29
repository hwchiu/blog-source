---
title: '[文章導讀] - 基於10G網路的 Kubernetes CNI 效能比較'
keywords: 'Kubernetes,Network,CNI,10G,Performance'
tags:
  - Kubernetes
  - CNI
  - Network
description: 本篇文章是節錄自網路上一篇關於 CNI 於10G網路下的效能分析，主要是讀後心得分享
abbrlink: 65302
date: 2020-09-17 21:29:53
---

# 前言

本文參考自 [Benchmark results of Kubernetes network plugins (CNI) over 10Gbit/s network (Updated: August 2020)](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-updated-august-2020-6e1b757b9e49)，主要用來記錄閱讀的心得分享，詳細全文請點選上述連結觀看。





## 環境

本篇文章的環境基於下列版本

1. Kubernetes: 1.19
2. Ubuntu: 18.04
3. 受測 CNI:
   1. Antrea v.0.9.1
   2. Calico v3.16
   3. Canal v3.16 (Flannel network + Calico Network Policies)
   4. Cilium 1.8.2
   5. Flannel 0.12.0
   6. Kube-router latest (2020–08–25)
   7. WeaveNet 2.7.0

內容是 2020 8月份時進行的實測結果



該文用到的所有測試工具全部都開源並放到 Github 上，對其專案有興趣的可以到這邊觀看內容 [benchmark-k8s-cni-2020-08](https://github.com/InfraBuilder/benchmark-k8s-cni-2020-08) 或是閱讀本文的第一大章節，有介紹一些工具的使用。



# MTU 的影響

文章中針對三款 CNI (Calico, Canal, WeaveNet) 測試看看 `偵測MTU的功能` 基於 TCP/UDP 下的效能如何

## TCP

| CNI\MTU   | Auto MTU | Custom MTU |
| --------- | -------- | ---------- |
| Calico    | 8876     | 9834       |
| Canal     | 8530     | 9817       |
| Weave Net | 4880     | 9759       |

以上數字都是 Mbit/s

## UDP

| CNI\MTU   | Auto MTU | Custom MTU |
| --------- | -------- | ---------- |
| 1Calico   | 1901     | 9757       |
| Canal     | 1820     | 9742       |
| Weave Net | 1809     | 9593       |

以上數字都是 Mbit/s

## 結論

上述的結論可以看到 `Auto MTU` 的效能都非常差，原因並不是 `Auto MTU` 沒有效，而是因為這些 CNI 目前根本沒有支持 `Auto MTU` 的做法，而 `Calico` 直到 3.7 版本才正式支持 `Auto MTU` 這個功能，而且根據作者的測試其功能良好。

作者認為對於這種需要設定 Jumbo frames 的環境下，如果沒有 `Auto MTU` 的話，管理員則需要手動去設定這些 MTU，所以非常希望每個 CNI 能夠去實作 `Auto MTU` 的功能來自動偵測並且設定，減少管理員需要人工介入的需求。



至於其他的 CNI 為什麼沒有測試，因為他們都有實作 Auto-MTU 的功能

1. Antrea
2. Cilium
3. Flannel
4. Kube-OVN

其中 `Kube-router` 這邊作者標示為 None, 估計可能是根本不能設定 MTU





# 使用 Raw Data 來進行 CNI 的評測

這章節主要會用來比較這些再正確 MTU 設定下不同 CNI 之間的效能比較



## 資源效能評比

一開始作者先比較基於閒置狀況下，不同 CNI 所消耗的資源狀況，包含了 CPU 以及 Memory。

原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 3    | 4      |
| Calico       | 3    | 4      |
| Canal        | 3    | 4      |
| Cilium       | 5    | 5      |
| Flannel      | 2    | 3      |
| Kube-OVN     | 8    | 4      |
| Kube-router  | 3    | 3      |
| Weave Net    | 2    | 3      |
| 裸機         | 1    | 2      |



這邊可以觀察到

1. Weave Net/Flannel 使用的效率最低，

2. Cilium 資源使用量偏高
3. Kube-OVN 資源使用量最高

4. 剩下的資源使用量都差不多，我認為可以算是同一個等級

> Kube-OVN > Cilium > 剩下全部 > WeaveNet/Flannel

## Pod to Pod

接下來看一下 Pod to Pod 的存取，這邊的方式是直接用 Pod 的 IP 來存去，並不是任何用 Service 這種方式來存取。

### TCP

| CNI\Performance | Mbit/s |
| --------------- | ------ |
| Antrea          | 9826   |
| Calico          | 9834   |
| Canal           | 9817   |
| Cilium          | 9426   |
| Flannel         | 9690   |
| Kube-OVN        | 9029   |
| Kube-router     | 8051   |
| Weave Net       | 9759   |
| 裸機            | 9906   |

從上面的數據可以觀察到

1. Kube-router 的數據最差
2. Kube-OVN 也沒有很好，大概就 9Gb/s 左右
3. Cilium 大概介於 9.5Gb/s
4. 剩下的都 CNI 效能跟裸機都不會差太多



接下來觀察一下這個實驗中，不同 CNI 的資源消耗量，原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 2    | 3      |
| Calico       | 2    | 3      |
| Canal        | 2    | 3      |
| Cilium       | 3    | 4      |
| Flannel      | 2    | 3      |
| Kube-OVN     | 4    | 4      |
| Kube-router  | 2    | 3      |
| Weave Net    | 3    | 3      |
| 裸機         | 1    | 2      |

從上面的結論觀察，我認為跟 `閒置` 的情況差不多，唯一的差異就是 Weavenet 從最少使用量的 CNI 變成第三高

> Kube-OVN > Cilium > WeaveNet > 剩下全部



### UDP

| CNI\Performance | Mbit/s |
| --------------- | ------ |
| Antrea          | 9796   |
| Calico          | 9757   |
| Canal           | 9742   |
| Cilium          | 9726   |
| Flannel         | 9846   |
| Kube-OVN        | 5833   |
| Kube-router     | 2810   |
| Weave Net       | 9539   |
| 裸機            | 9885   |

從上面的數據可以觀察到

1. Kube-router 的數據最差，連 3Gb/s 都不到，非常慘，不到 30% 的效能
2. Kube-OVN 也很不好，大概只有 6Gb/s
3. 剩下的都 CNI 效能跟裸機都不會差太多



接下來觀察一下這個實驗中，不同 CNI 的資源消耗量，原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 4    | 4      |
| Calico       | 3    | 4      |
| Canal        | 3    | 4      |
| Cilium       | 4    | 5      |
| Flannel      | 3    | 4      |
| Kube-OVN     | 5    | 5      |
| Kube-router  | 4    | 4      |
| Weave Net    | 4    | 4      |
| 裸機         | 2    | 2      |

從上面的結論觀察，跟 `閒置` 比較起來比較有一些變化

1. Kube-OVN 永遠都是使用資源第一名
2. Cilium 還是第二名
3. 第三名則是 WeaveNet/Antrea/Kube-Router
4. 剩下的等級差不多

Kube-OVN > Cilium > WeaveNet/Antrea/Kube-Router > Calico/Canal/Flannel > 裸機



## Pod to Service

這個情況下則是探討透過 Service 來存取目標 Pod，也是基於 TCP/UDP 來測試，其中 Service 則是基於 ClusterIP 的設定才測試。

### TCP

| CNI\Performance | Mbit/s |
| --------------- | ------ |
| Antrea          | 9789   |
| Calico          | 9841   |
| Canal           | 9757   |
| Cilium          | 9630   |
| Flannel         | 9826   |
| Kube-OVN        | 9409   |
| Kube-router     | 8380   |
| Weave Net       | 9749   |

從上面的數據可以觀察到

1. Kube-router 的數據最差, 大概只剩下 85% 效能
2. Kube-OVN 還行，大概 95%
3. 剩下的都 CNI 效能都差不多， 97% +-1%。



接下來觀察一下這個實驗中，不同 CNI 的資源消耗量，原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 3    | 4      |
| Calico       | 2    | 4      |
| Canal        | 2    | 4      |
| Cilium       | 3    | 5      |
| Flannel      | 2    | 4      |
| Kube-OVN     | 4    | 5      |
| Kube-router  | 2    | 4      |
| Weave Net    | 3    | 4      |

從上面的結論觀察，跟 `閒置` 比較起來比較有一些變化

1. Kube-OVN 永遠都是使用資源第一名
2. Cilium 還是第二名
3. 第三名則是 WeaveNet/Antrea
4. 剩下的等級差不多

Kube-OVN > Cilium > WeaveNet/Antrea > Kube-Router/Calico/Canal/Flannel > 裸機

相對於 Pod to Pod 的情況來看， Pod to Service 中 Antrea 的效能消耗更高，從第四名那群躍升到第三名。



### UDP

| CNI\Performance | Mbit/s |
| --------------- | ------ |
| Antrea          | 8618   |
| Calico          | 9420   |
| Canal           | 9800   |
| Cilium          | 9712   |
| Flannel         | 9825   |
| Kube-OVN        | 5380   |
| Kube-router     | 2781   |
| Weave Net       | 9154   |

從上面的數據可以觀察到

1. Kube-router 的數據最差，連 3Gb/s 都不到，非常慘，不到 30% 的效能
2. Kube-OVN 也很不好，大概只有 5Gb/s
3. Antrea 的效能也不好了，大概只有 8.6 Gb/s
4. Calico 以及 WeaveNet 的效能也都降到 95% 以下，不到 9.5Gb/s
5. 剩下的都 CNI 效能都差不多 (Canal/Cilium/Flannel)



接下來觀察一下這個實驗中，不同 CNI 的資源消耗量，原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 4    | 4      |
| Calico       | 3    | 4      |
| Canal        | 3    | 4      |
| Cilium       | 4    | 5      |
| Flannel      | 3    | 4      |
| Kube-OVN     | 4    | 5      |
| Kube-router  | 3    | 4      |
| Weave Net    | 4    | 4      |

從上面的結論觀察，跟 `閒置` 比較起來比較有一些變化

1. Kube-OVN 跟 Cilium 兩個各有千秋，一個 CPU 用比較多，一個則是記憶體比較多
2. Antrea/WeaveNet/Kube-router 三者消耗的層級差不多
3. Calico/Canal/Flannel 三者差不多

Kube-OVN/Cilium > WeaveNet/Antrea/Kube-Router > Calico/Canal/Flannel



## Network Policies

這邊沒有任何的數據測試，除了 Flannel 外，剩下的 CNI 都有實現 Ingress/Egress(往內/往外) 的 Network Policies，很棒!

# CNI 加密

測試的 CNI 解決方案中，只有四套有支援資料加密的部分，分別是

1. Antrea (IPSec)
2. Calico (wireguard)
3. Cilium (IPSec)
4. WeaveNet (IPSec)



## 效能部分

因為這邊效能比較少，所以我們將 TCP/UDP 放在一起評比

## Pod to Pod TCP/UDP

| CNI\Performance | TCP (Mbit/s) | UDP (Mbit/s) |
| --------------- | ------------ | ------------ |
| Antrea          | 1392         | 742          |
| Calico          | 9786         | 4877         |
| Cilium          | 1657         | 869          |
| WeaveNet        | 1384         | 432          |

這邊可以觀察到

1. TCP 的效能遠勝於 UDP
2. 使用 wireguard 的效能完全輾壓 IPSec 技術的其他 CNI
3. 三個都使用 IPSec 的 CNI，其中 WeaveNet 效能是其中最差的，而 Cilium 則是效能最好的



## 資源效能評比 TCP/UDP



### TCP

接下來觀察一下這個實驗中，不同 CNI 的資源消耗量，原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 2    | 4      |
| Calico       | 4    | 4      |
| Cilium       | 2    | 5      |
| WeaveNet     | 2    | 4      |

從上面的結論觀察，跟 `閒置` 比較起來比較有一些變化

1. Calico 使用的資源最多
2. 剩下三者差不多

### UDP

接下來觀察一下這個實驗中，不同 CNI 的資源消耗量，原文中是用 CPU(%) 以及 Memory (MB) 來畫圖，我則是將這些數字用幾個等級來區分，`數字愈低代表使用量愈低`

| CNI\資源類型 | CPU  | Memory |
| ------------ | ---- | ------ |
| Antrea       | 2    | 4      |
| Calico       | 4    | 4      |
| Cilium       | 2    | 5      |
| WeaveNet     | 2    | 4      |

從上面的結論觀察，跟 `閒置` 比較起來比較有一些變化

1. Calico 使用的資源最多
2. 剩下三者差不多



## Pod to Service TCP/UDP

| CNI\Performance | TCP (Mbit/s) | UDP (Mbit/s) |
| --------------- | ------------ | ------------ |
| Antrea          | 1390         | 735          |
| Calico          | 9808         | 4872         |
| Cilium          | 1661         | 866          |
| WeaveNet        | 1381         | 451          |

這邊可以觀察到其結果與 Pod-to-Pod 是差不多的，因此結論完全一致

1. TCP 的效能遠勝於 UDP
2. 使用 wireguard 的效能完全輾壓 IPSec 技術的其他 CNI
3. 三個都使用 IPSec 的 CNI，其中 WeaveNet 效能是其中最差的，而 Cilium 則是效能最好的







# 總結



根據上述的全部來源，我們可以繪製數張總結表格，效能的部分採用相對比較，對原始數字有興趣的可以參考其公開的 Github 專案。

評比標準: `好>普通>不好`

## MTU/加密效果

| ---         | MTU設定  | Network Policy(往內) | Network Policy(往外) | 加密設定         | 加密後傳輸效能 | 加密資源消耗量 |
| ----------- | -------- | -------------------- | -------------------- | ---------------- | -------------- | -------------- |
| Antrea      | 自動偵測 | 支援                 | 支援                 | 支援(部署時設定) | 普通           | 普通           |
| Calico      | 手動設定 | 支援                 | 支援                 | 動態設定         | 好             | 不好           |
| Canal       | 手動設定 | 支援                 | 支援                 | 不支援           | n/a            | n/a            |
| Cilium      | 自動偵測 | 支援                 | 支援                 | 支援(部署時設定) | 普通           | 普通           |
| Flannel     | 自動偵測 | 不支援               | 不支援               | 不支援           | n/a            | n/a            |
| Kube-OVN    | 自動偵測 | 支援                 | 支援                 | 不支援           | n/a            | n/a            |
| Kube-router | 無       | 支援                 | 支援                 | 不支援           | n/a            | n/a            |
| Weave Net   | 手動設定 | 支援                 | 支援                 | 支援(部署時設定) | 普通           | 普通           |



## 流量評比

這邊使用一些縮寫

P2P -> Pod to Pod

P2S -> Pod to Service

| ---         | P2P/TCP | P2P/UDP | P2S/TCP | P2S/UDP |
| ----------- | ------- | ------- | ------- | ------- |
| Antrea      | 很好    | 很好    | 很好    | 普通    |
| Calico      | 很好    | 很好    | 很好    | 好      |
| Canal       | 很好    | 很好    | 很好    | 很好    |
| Cilium      | 好      | 很好    | 很好    | 很好    |
| Flannel     | 很好    | 很好    | 很好    | 很好    |
| Kube-OVN    | 好      | 很差    | 好      | 很差    |
| Kube-router | 很差    | 很差    | 很差    | 超級差  |
| Weave Net   | 很好    | 很好    | 很好    | 好      |

## 資源消耗評比

這邊使用一些縮寫

P2P -> Pod to Pod

P2S -> Pod to Service

同時評比的概念是使用的資源多寡，採用相對等級

超高>有點高>普通>少

| ---         | 閒置   | P2P/TCP | P2P/UDP | P2S/TCP | P2S/UDP |
| ----------- | ------ | ------- | ------- | ------- | ------- |
| Antrea      | 普通   | 普通    | 普通    | 普通    | 普通    |
| Calico      | 普通   | 普通    | 少      | 少      | 少      |
| Canal       | 普通   | 普通    | 少      | 少      | 少      |
| Cilium      | 有點高 | 超高    | 有點高  | 有點高  | 超高    |
| Flannel     | 少     | 普通    | 少      | 少      | 少      |
| Kube-OVN    | 超高   | 超高    | 超高    | 有點高  | 超高    |
| Kube-router | 普通   | 普通    | 普通    | 少      | 普通    |
| Weave Net   | 少     | 有點高  | 普通    | 普通    | 普通    |



透謝圖表可以觀察到

1. Kube-OVN 不但資源吃很多，效能還很不好
2. Canal/Calico/Flannel 三者的運算資源使用量都不多，且效能都很好
3. Kube-Router 的效能都很差，資源使用方便也不是特別出色
4. WeaveNet 與 Cilium 效能都不差，但是 Cilium 吃的效能很高，可說跟 Kube-OVN 同等級，而 WeaveNet 用到的資源少



# 個人心得

1. 這次的實驗評比我認為其實能看到的東西有限，主要是不同的 CNI 所搭配的解決方案不同，目標要配合的情境也不同，雖然從圖表中可以看到 Kube-OVN 的綜合評比最差，但是其要用的場景本身就不太一樣，單純用最原始的流量互打來判別優劣其實不太對
2. 如果今天要選擇網路 CNI 的時候，可以看到效能跟資源方面， Flannel/Calico/Canal 幾乎等級都差不多，而且 Calico 還支援加密與 Network Policy 等功能。
3. 此外，目前 Flannel 也從 Kubeadm 的官方教學頁面中移除，因為太多問題且維護者沒有要修復。所以我認為如果沒有特別使用情境需求的話，可以考慮使用 Calico.
4. Cilium 對於安全性以及 load-balancing 方面也有別的功能，就如同(1)點所提到，不同的場景有不同的需求，有些功能是獨占的。





# 參考來源

- https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-updated-august-2020-6e1b757b9e49

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
