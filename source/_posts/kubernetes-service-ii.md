---
title: '[Kubernetes] How To Implement Kubernetes Service(i)'
keywords: 'Kubernetes,Service,Linux,k8s,iptables'
abbrlink: 3141
date: 2018-08-22 00:56:15
tags:
  - Kubernetes
  - Linux
  - Iptables
description:
---

本文章是屬於 `kubernetes` service 系列文之一，該系列文希望能夠與大家討論下量兩個觀念
1. 什麼是 `Kubernetes Service`, 為什麼我們需要它？ 它能夠幫忙解決什麼問題
2. `Kubernetes Service` 是怎麼實現的?， 讓我們用 iptables 來徹徹底底的理解他

相關文章:
[[Kubernetes] What is Service](https://www.hwchiu.com/kubernetes-service-i.html)


本篇文章著重於後者，透過對系統上的分析來探討 `kubernetes service` 實作的原理。
由於篇幅有限，本文會將基本概念都說明一遍，並且透過 `ClusterIP` 這個形式來解釋其運作原理。
待下篇文章在來解釋 `NodePort` 以及 `SessionAffinity` 是如何實現的。



很多人在學習一個新的系統的時候，最初接觸的都是如何使用，如何操作，然而對其背後的實現原理卻沒有太多的著墨。
為什麼要學習實現的原理？ 
很簡單，因為「知己知彼，百戰百勝」
當我們理解這些功能背後的實現方式後，我們不但可以學習到其背後的設計理念與方式，同時當問題出現時。
我們可以比一般使用者用更廣的角度去思考問題，同時也可以採用更系統的方式去除錯找出問題的根本，這更能體現出你相對於一般人的價值。



## Introduction

在前篇文章 [[Kubernetes] What is Service](https://www.hwchiu.com/kubernetes-service-i.htmlvvv) 我們已經學習到關於 `Kubernetes Service` 的基本概念與用法
而本篇文章則是想要探討在預設安裝情況下， `kubernetes` 是如何實現 `service` 的功能。

<!--more-->

## Kube-Proxy Mode
目前的 `Kubernetes` 裡面總共有三種方法去實現 `Service`，分別是
1. kube-Proxy (old)
2. iptables (default)
3. ipvs (experimental)
 
簡單敘述一下這三種的差異
1. kube-Proxy 是指透過`kube-proxy`本身程式內部的邏輯來實現 `service`，由於 `kube-proxy` 是 `user-space` 的應用程式，所以效率非常低落，但是因為是程式化的結果，彈性比較高。
2. iptables 的話，則是讓 `kube-proxy` 去維護跟 `service`有關的邏輯部分，真正所有封包轉送都交由 `kernel-space` 的 `iptables` 來處理。 效率比(1)來得強，但是在使用上則是會受限於 `iptables` 的規則與框架。
3. ipvs(IP virtual switch) 與(2)類似，只是在 `kernel-space` 裡面採用 `ipvs` 的方式來轉送封包，相對於 `iptables` 本身效率更高，同時也不會受限於`iptables` 的使用規則


我們可以藉由設定 `kube-proxy` 裡面的 `--proxy-mode` 這個參數來決定要使用哪一種實現方式

{% note danger %}
kube-proxy 本身會透過 `daemonset` 的方式部屬到每一個節點上，有興趣的可以透過
kubectl -n kube-system describe ds kube-proxy 指令觀察一下相關的內容
{% endnote %}

回歸正題，本文主要探討的對象是 `iptables`，看看這個歷史悠久且功能強大的 `iptables` 框架是如何完成 `kubernetes service` 所需要的各種功能

## Iptables
`iptables` 本身真正講起來，其實是透過 `user-space` 的管理工具`iptables` 搭配 `kernel-space` 的 `netfilter` 網路子系統兩者組合來提供各式各樣的功能。

想要更加瞭解 `iptables` 的介紹可以參閱我在 [COSCUP x GNOME.Asia x openSUSE.Asia 2018](https://www.slideshare.net/hongweiqiu/understand-the-iptables-step-by-step-109650841) 所講授的透過閱讀原始碼的方式來更加瞭解 `iptables`.
此外我之後也會撰寫跟 `iptables` 相關的文章，到時候會整理相關的內容讓大家更容易去理解這套強大的工具。

為了理解本篇文章接下來的內容，我們必須要對 `iptables` 有一些基本的認識，這邊幫大家整理一下需要的概念。
1. 在 `iptables` 裡面每個規則面對的對象都是一個個的網路封包
2. 每條規則的邏輯大抵上如下
    - 請問該封包有沒有符合 `Match` 條件
    - 如果有 `Match`, 則執行目標 `Action(Target)`
4. 每條規則都屬於一個 `chain`, 一個 `chain` 可以有多條規則
5. 除了內建的 `chain` 之外, 管理者可以自定義新的 `chain` 方便管理

有了基本概念後，我們就可以開始來探討 `kubernetes serivce` 如何實作了

{% note danger %}
若需要真正完全理解這一切，還需要搭配 `PREROTUING`,`FORWARD`,`POSTRING` 等不同 `table` 的先後關係與概念，不但對於理解有更加的幫助，對於除錯找問題也是有很大的效益。
有興趣的讀者可自行上網尋找相關文章學習，或是等我哪天會寫出相關的文章來介紹這些資訊。
{% endnote %}


## Kubernetes Service
在我們開始前，我們要先定義幾個相關的名詞，方便之後閱讀的時候可以順利的理解前後文的關係與概念。

這邊先借用 [[Kubernetes] What is Service](https://www.hwchiu.com/kubernetes-service-i.htmlvvv) 內最後展示範例使用的概念圖
![Imgur](https://i.imgur.com/osNqxlw.png)


### Endpoints
在 `kubernetes` 內有一個名為 `endpoints` 的資源，其代表的是 `service` 所關注目標服務實際上真正運行`Pod`的 `IP` 地址
```bash=
vortex-dev:02:00:28 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl get endpoints
NAME                ENDPOINTS                                      AGE
k8s-nginx-cluster   10.244.0.88:80,10.244.0.89:80,10.244.0.90:80   9h
k8s-nginx-node      10.244.0.88:80,10.244.0.89:80,10.244.0.90:80   9h
kubernetes          172.17.8.100:6443                              11d
```

可以從上述的輸出結果看到每個 `service` 都會對應到多組的 `endpoints`，所以當叢集內的容器有任何 `IP` 更動的時候，這邊的數據都會自動更新，以確保 `service` 有辦法存取後端真正的服務

{% note danger %}
有在使用 `Service` 的讀者，以後若有遇到 `service` 不通的情況，可以嘗試先看看該 `service` 是否有對應的 `endpoints`，沒有的話可能是 `selector` 寫錯或是目標服務根本沒有運行起來。
{% endnote %}

### Custom Chain
`kubenetes` 使用 iptables 時為了更有效管理不同的功能與規則的歸屬，建立的大量的 `custom chain`
### iptables-save
這是一個 `iptables` 相關的指令，我個人很喜歡用它來觀察 `iptables` 的規則，本文的所有範例都會是使用該指令進行展示

## ClusterIP

我們已經知道 `ClusterIP` 的作用範圍只有`叢集內`的應用程式/節點，所以在本段落我們會著重於三個概念來理解

{% note danger %}
叢集內節點是的存取比較尷尬，的確可以透過 `ClusterIP` 地址來存取，但是預設情況下是沒有辦法解析`FQDN`取得對應的 `ClusterIP` 地址。
{% endnote %}

1. 如何透過 `FQDN` 輾轉存取到目標容器們(Endpoints)
2. 如何做到只有`叢集內`的應用程式/節點才可以存取
3. 假設有多個目標容器(Endpoints), 這中間的選擇方式是怎麼處理?

### Access By FQDN
我們都知道 `Service` 本身會提供一組對應的 `FQDN` 供應用程式使用
實際上這組`FQDN` 只有 `kube-dns` 能夠理解，而且其對應的 `IP` 地址其實就是每個 `Service` 提供的 `ClusterIP` 
{% note dnager  %}
這邊的ClusterIP剛好跟 `Type` 的ClusterIP 名稱，但是這邊要表示的真的是個`IP`地址
{% endnote %}

```bash=
vortex-dev:05:36:40 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl get svc
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
k8s-nginx-cluster   ClusterIP   10.98.51.150   <none>        80/TCP         1d
k8s-nginx-node      NodePort    10.99.157.45   <none>        80:32293/TCP   1d
```

在此範例中，可以看到不論是 `ClusterIP` 或是 `NodePort` 實際上都會有一組 `Cluster-IP` 的 `IP` 地址。

這個`Cluster-IP`最大的特性就是他是一個虛擬的`IP`地址，在整個`kubernetes`叢集內是找不到任何一張網卡擁有這個`IP`地址的。

所有針對該`Cluster-IP`發送的封包，在滿足特定的條件下，都會被透過`DNAT(Destination Network Address Translation)` 進行轉換，在`service`  其實就會是被轉換到其中一個 `EndPoints` 的真正 `IP` 地址

```bash=
vortex-dev:05:43:50 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$sudo iptables-save -t nat | grep nginx-cluster | grep DNAT
-A KUBE-SEP-7MBJVYFMXTKOJUKD -p tcp -m comment --comment "default/k8s-nginx-cluster:" -m tcp -j DNAT --to-destination 10.244.0.88:80
-A KUBE-SEP-ARZAHNE3T3EMMTGB -p tcp -m comment --comment "default/k8s-nginx-cluster:" -m tcp -j DNAT --to-destination 10.244.0.90:80
-A KUBE-SEP-O3CWA7STMVCKFPRY -p tcp -m comment --comment "default/k8s-nginx-cluster:" -m tcp -j DNAT --to-destination 10.244.0.89:80

vortex-dev:05:43:54 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl get endpoints k8s-nginx-cluster
NAME                ENDPOINTS                                      AGE
k8s-nginx-cluster   10.244.0.88:80,10.244.0.89:80,10.244.0.90:80   1d
```

透過 `iptables` 的觀察，我們可以看到在某些 `custom chain` 中會透過 `DNAT` 的方式把封包的目標`IP` 位址轉換到這些`endpoints` 擁有的`IP`地址。
實際上這個 `custom chain` 就是 `KUBE-SEP-XXXX`, 每個 `Endpoints` 都有一條屬於自己的 `custom chain`. 而 `KUBE-SEP` 我想其含意應該就是 `KUBE Service Endpoint` 吧。

剛剛我們對`iptables`規則的理解，每個規則都是`符合條件`,`做一件事情`，因此背後有多少個`endpoints`,實際上就會有多少條規則在處理`DNAT`。

到這邊我們還有一些疑問還沒有解開，只要先記住下面的結論就好。

{% note info %}
1. 每個`service`的`FQDN`都會對應到一組`Cluster-IP` `IP` 地址，該地址其實是虛擬`IP`地址。
2. 送往該`Cluster-IP` 的封包在滿足特定的情況下，會透過`DNAT`的方式轉換成其中一個`endpoints`容器上的真實`IP`地址
{% endnote %}


### Cluster Only
現在我們要來討論一下，到底所謂的只有`叢集內`的應用程式/節點才可以存取`clusterIP`這到底是怎麼運作的。

我們複習一下前面的某個敘述
{% note danger %}
送往該`Cluster-IP` 的封包在滿足特定的情況下，會透過`DNAT`的方式轉換成其中一個`endpoints`容器上的真實`IP`地址
{% endnote %}

這邊提到要滿足特定的情況才會走到`DNAT`轉到對應的`EndPoints`。
所以
**只有`叢集內`的應用程式/節點才可以存取** 其實就是 **特定的情況**

我們都知道 `iptables` 的規則可以根據封包的一些資訊來做比對，所以我們能不能做出一種規則是
只有 **封包的來源`IP`地址是來自`叢集內的應用程式/節點`**，符合這種規則的才有資格去進行 `DNAT` 進行轉發

實際上使用的概念是更簡單，這邊透過 `iptables build-in chain` 裡面的 `OUTPUT/PREROUTING` 兩個 `chain` 來達成
**只有`叢集內`的應用程式/節點** 
這個功能

這邊我直接講明
- OUTPUT: 本地節點送出去的封包都會先走到這邊
- PREROUTING: 本地網卡收到封包後會走到這邊，包含了`Contaienr`出來的封包都會走到

接下我們來透過 `iptables` 的指令來觀察一下這些規則。

根據前面的查詢，我們知道 `ClusterIP` 地址是 `10.98.51.150`,
```bash=
vortex-dev:04:24:49 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$sudo iptables-save | grep k8s-nginx-cluster
....
-A KUBE-SERVICES -d 10.98.51.150/32 -p tcp -m comment --comment "default/k8s-nginx-cluster: cluster IP" -m tcp --dport 80 -j KUBE-SVC-3FL7SSXCKTCXAYCR
....
```
上述的規則我們來仔細看一下每個參數的意義
- -A `KUBE-SERVICES` 
    - 這是一個 `Custom Chain`, 所有跟 `Kubernetes Service` 有關的第一到防線規則都在這邊
- -d 10.98.51.150/32
    - 目標位置是 `ClusterIP` 的話
- -p tcp
    - 目標是 TCP 協定
- -m comment
    - 就是註解
- -m tcp --dport 80
    - 使用外掛模組來解析TCP裡面的資訊，希望 TCP port 是80
- -j KUBE-SVC-3FL7SSXCKTCXAYCR
    - 上述所有條件都符合，就會跳入另外一個`custom chain`來執行後續任務

後面的部份我們先不管他，我們來看一下什麼情況下的封包會進入到 `KUBE-SERVICES` 這個 `custom chain`.

```bash=
$sudo iptables-save -c | grep KUBE-SERVICES
:KUBE-SERVICES - [0:0]
[2376:171145] -A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
[3706:223392] -A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
...
```

這邊可以看到有兩條規則，分別對應到原生的 `OUTPUT` 以及 `PREROUTING`，直接透過 `-j` 直接跳入到 `KUBE-SERVICES` 來進行後續處理。

{% note danger %}
其實夠熟悉 iptables 的朋友應該已經可以猜到，在此規則狀況下，我只要有辦法讓流向`ClusterIP`的封包透過一些網路規則的方式流向到叢集內的節點，依然可以順利的存取背後的服務。
只是因為這些`ClusterIP`本身不存在網路之中，所以需要針對整個網路的路由表規則額外設定
這部份就是額外有興趣的人可以自己研究，這邊就不再多敘述。

{% endnote %}

我們先用下圖來幫助目前的概念做一個整理

- 橘色底的代表是封包的來源，在此案例中其實就代表`叢集內的節點/應用程式`
- 綠色底代表的是`iptables build-in chain`，主要用來處理叢集內應用程式/節點上的封包傳輸
- 藍色的則是`kubernetes` 的 `custom chain`.
- 紫色的則是代表 `iptables` 的描述規則
- 紅色則是我們知道最後會在 `KUBE-SEP-XXX` 透過 `DNAT` 把封包轉換到其中一個`endpoints`之中。
- ???則是代表我們還沒有研究到的，只知道這中間還有一部分的謎團等待解開

![Imgur](https://i.imgur.com/74jQEiM.png)

{% note info %}
1. 每個`service`的`FQDN`都會對應到一組`ClusterIP` `IP` 地址，該地址其實是虛擬`IP`地址。
2. 透過 `iptablse` 的 `OUTPUT/PREROUTING`，其有能力去匹配所有叢集內的應用程式/節點所送出的封包
3. 最後透過去比對封包的目的地位址是否是 `ClusterIP` 來決定要不要往下跳到其他`custom chain` 去處理。
4. 封包最後會透過`DNAT`的方式轉換成其中一個`endpoints`容器上的真實`IP`地址
{% endnote %}


### Loab Balancing 
現在我們要來看看最後一個部分了，到底要怎麼從眾多的 `Endpoints` 中挑選出一個可用的 `Pod` 來使用。

根據前面的分析，當我們的封包符合叢集內使用的規則後，會跳到一個`KUBE-SVC-3FL7SSXCKTCXAYCR` 的 `custom chain`.
實際上 `KUBE-SVC-XXXX` 的 `custom chain` 就是用來處理挑選 `Endpoints` 用的，會根據每個 `kubernetes service` 創造一條屬於其的 `chain`.

我們先重新認真看一下這條規則
```bash=
-A KUBE-SERVICES -d 10.98.51.150/32 -p tcp -m comment --comment "default/k8s-nginx-cluster: cluster IP" -m tcp --dport 80 -j KUBE-SVC-3FL7SSXCKTCXAYCR
```

當封包滿足叢集內的條件時，就會跳到一個名為`KUBE-SVC-3FL7SSXCKTCXAYCR`的 `custom chain`.

這時候來仔細檢視其內容

```bash=
-A KUBE-SVC-3FL7SSXCKTCXAYCR -m comment --comment "default/k8s-nginx-cluster:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-POVAFWTN5ECIRK7J
-A KUBE-SVC-3FL7SSXCKTCXAYCR -m comment --comment "default/k8s-nginx-cluster:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-AQWRPA7WRPWQAWLR
-A KUBE-SVC-3FL7SSXCKTCXAYCR -m comment --comment "default/k8s-nginx-cluster:" -j KUBE-SEP-XPSDT7KEI65EZ2WI
```

我們可以觀察到裡面有幾個重點
1. 有 `-m statistic`, `random`, `probability` 這些跟機率相關的文字。
2. 滿足特定條件後，都會跳到 `KUBE-SEP-XXXXX` 這些 `custom chain`. 這就如同我們之前所觀察到會執行 `DNAT` 的 `custom chain` 了

接下來說明一下到底那群跟機率有關的規則是怎麼運作的。
我們先前已經說明過，`iptables`的運作方式是**符合條件**, **就做一件事情**
所以並沒有很簡單的在一條規則內，幫你選出對應的**Endpoints**.
於是這邊的作法是，假設我有三個 **Endpoint**，挑選的流程如下
1. 請問機率大神，給我一個數字
    - 若該數字<0.33, 則使用第一個**endpoints**
    - 否則重新問機率大神，從剩下的 **endpoints** 挑選
3. 請問機率大神，再次給我一個數字
    - 若該數字<0.5, 則使用第二個 **endpoints**
    - 否則直接使用地三個 **endpoints**.

用下圖的方式來重新解釋這個流程，假設今天有四個 `Endpoints` 要選擇
![Imgur](https://i.imgur.com/tpSFwKg.png)
1. 一開始要從四個裡面選擇，所以機率只有 **1/4**, 若符合了就採用第一個 `Endpoint`
2. 因為前面沒有符合，所以接下來要從三個裡面繼續選擇下一個 **endpoints**，這時候的機率就是**1/3**,但是因為要走到這步必須(1)沒有成功，所以機率是(3/4*1/3), 就是 **1/4**
3. 一此類推，每個 **Endpoints** 的機率都是 **1/4**
4. 運氣最不好的 **endpoints** 必須要進行 **n-1** 次的規則比對 (n是endpoints的數量)
5. 運氣最好的只需要一次比對就可以找到。

當找到要使用的 **Endpoints** 的時候，就會跳到對應的 **KUBE-SEP-XXXX** 去進行 **DNAT** 的轉換。


## Summary

最後一塊拼圖也已經完成了，到這邊已經可以大概知道是如何透過 `iptables` 來完成 `clusterIP` 的轉發。
在這種實作架構中，每個節點的 `iptables` 都會自行去負責尋找 `endpoints` 來處理，而`ClusterIP` 這個不存在的`IP`地址只是幫助我們讓`iptables`有個好依據來處理。

就用下圖來幫這篇文章做個最後的結尾。
下篇文章在來仔細看看 `NodePort` 以及 `SessionAffinity` 這些功能如何透過 `iptables` 來實現。

![Imgur](https://i.imgur.com/xoPvipq.png)

