---
title: '[Kubernetes] How to Implement Kubernetes Service - SessionAffinity'
keywords: 'Kubernetes,Service,Linux,k8s,iptables,sessionaffinity,session'
abbrlink: 60380
date: 2018-08-27 13:46:47
tags:
  - Kubernetes
  - Linux
  - Iptables
description: 在前述中我們已經學過了什麼是 kubernetes service 以及如何實現 ClusterIP/NodePort 等 service 類型. 透過對 iptables 的探討與研究. 接下來要研究的則是 Service 裡面的一個參數, 叫做 SessionAffinity, 所謂的連線親和力, 透過該參數的設定,希望能夠讓符合特定條件的連線最後都會選用到相同的後端應用程式,目前有支援的選項是 ClinetIP, 這意味者只要連線的來源 IP 地址是相同的,最後都會被導向到相同的容器應用程式. 然而這部分到底是如何實現的, 本文會先介紹什麼叫做 Connection. 並且介紹 SessionAffinity 的使用方式以及使用後的結果. 最後一樣透過相同的思路, 藉由研究 iptables 的規則來學習到 SessionAffinity 要如何完成, 同時也可以學習到 iptables 衆多靈活的使用方式.

---

# Preface

本文章是屬於 `kubernetes` service 系列文之一，該系列文希望能夠與大家討論下量兩個觀念
1. 什麼是 `Kubernetes Service`, 為什麼我們需要它？ 它能夠幫忙解決什麼問題
2. `Kubernetes Service` 是怎麼實現的?， 讓我們用 iptables 來徹徹底底的理解他

相關文章:
[[Kubernetes] What is Service](https://www.hwchiu.com/kubernetes-service-i.html)
[[Kubernetes] How To Implement Kubernetes Service - ClusterIP](https://www.hwchiu.com/kubernetes-service-ii.html)
[[Kubernetes] How To Implement Kubernetes Service - NodePort](https://www.hwchiu.com/kubernetes-service-iii.html)


本文銜接前續文章，繼續透過對 `iptables` 的分析來研究 `kubernetes service` 中 `SessionAffinity` 的實作原理。

# What Is Connection
在我們開始討論 `SessionAffinity` 之前，我們要先來討論一下，什麼叫做 `Connection`, 並且透過對 `Connection` 的瞭解，我們會更容易的理解到底 `SessionAffinity`  想要解決的問題。

這邊我們使用下列的圖示來說明到底什麼是 `Connection`

![Imgur](https://i.imgur.com/C18G0Ym.png)

這個範例中，我們有一個 `kubernetes` 叢集，其透過 `kubernetes service(NodePort)` 的方式將內部的 `Nginx` 服務讓外界能夠存取。

此時，在外部我們有兩台機器想要存取這些 `Nginx` 服務，其中第一台機器 **Host1** 上面有一個 `Client Application`, 透過 `kubernetes service` 的功用，其最後存取到了`Nginx1` 這個容器。
而第二台機器 **Host2** 上面則有兩個 `Client Application`, 其分別對應到的是 `Nginx2` 以及 `Nginx3` 這兩個容器。

首先，本文中所有提到的 `Connection` 代表的就是如上圖所示的 `ClientApp` 到 `Nginx` 的連線，所以上述的範例總共有三個 `Connection`.

這邊我們先思考一個問題，針對圖中 `ConnectionA` 來討論一下，該 `ConnectionA` 中所有的來回請求回應，都會走到 `Nginx1` 嘛?
這些所謂的請求回應，實際是都是網際網路世界中的封包，每個封包一旦到達 `Kubernetes ` 叢集內就會遇到 `iptaables` 規則的處理。
在這個情況下，我們如何保證該 `ConnectionA` 裡面的所有封包都可以送達到同一個 `Nginx1` 也就是所謂的 `EndPoints`.

這個問題其實已經解決了，是透過 `iptables` 解決的。
這個原理沒有辦法一言兩語解決，我之後若有時間會寫相關的文章介紹這邊的原理與實現。
這邊只要知道其方法是倚賴 `Linux Kernel` 裡面相關的技術去提供類似 `Cache` 的機制，確保相同的 `Connection` 內所有的來回封包都會執行相同的 `SNAT/DNAT`.

有興趣的讀者可以使用下列關鍵字去搜尋相關文章，不然就是等我哪天有時間在來仔細介紹這邊的概念lol
1. netfilter
2. conntrack
3. DNAT/SNAT

# What Is SessionAffinity
假如相同 `Connection` 內的封包都已經可以保證連接到相同的 `Endpoints` 了，那到底什麼是 `SessionAffinity`?

換個角度來說，我們有沒有辦法讓建立的新 `Connection` 都連接到相同的 `Endpoints` ?

這個問題就是 `SessionAffinity` 想要解決並處理的，透過一種機制，讓不同的 `Client Application` 所建立新的 `Connection` 最後都會連接到相同的 `EndPoints`.

以上圖的範例來說，有沒有可能讓 `Connection A,B,C` 都連接到相同的 `EndPoints`?

# Configuration
首先，我們先來看一下 `Kubernetes` 裡面定義的 `SessionAffinity` 要怎麼使用
目前總共有兩種類型可以選擇
1. None
2. ClientIP

第一種 `None` 其實就是什麼都不做，針對每一條新的`Connection`都不去採去任何動作，基本上就是每條`Connection`都看運氣來選擇最後選擇到哪一個 `Endpoints`.

第二種 `ClientIP` 則是真的有事情要做了，就如同字面上的意思，`Client IP`.
目的很簡單，對於每一條新建立的 `Connection`, 若其 `Client IP` 地址相同，就導到相同的 `Endpoints` 去使用。

所以回到剛剛上述的問題
以上圖的範例來說，有沒有可能讓 `Connection A,B,C` 都連接到相同的 `EndPoints`?
按照目前 `kubernetes` 的設計，上述的答案是在最簡單的網路架構下，只能夠確保 **Host2** 上面所建立的所有 **Connection** 都可以連接到相同的 **Endpoints**.

這邊假設這些 Host 都有自己的公開 `IP` 地址，不考慮任何 `SNAT` 的效果。

# How It Works

## Setup
接下來，我們繼續使用[kubeDemo](https://github.com/hwchiu/kubeDemo)來進行相關的服務部屬以及`iptables`規則研究。

```bash=
vortex-dev:01:37:28 [~/go/src/github.com/hwchiu/kubeDemo/services](master)vagrant
$kubectl apply -f service/nginx-
nginx-affinity.yml  nginx-cluster.yml   nginx-node.yml
vortex-dev:01:37:28 [~/go/src/github.com/hwchiu/kubeDemo/services](master)vagrant
$kubectl apply -f service/nginx-affinity.yml
service/k8s-nginx-affinity created
```

我們用下列指令確認一下剛剛部屬的 `kubernetes service` 是否真的有設定 `sessionAffinity`
```bash=
vortex-dev:01:40:58 [~/go/src/github.com/hwchiu/kubeDemo/services](master)vagrant
$kubectl get service k8s-nginx-affinity -o jsonpath='{.spec.sessionAffinity}'
ClientIP
```

## IPTABLES
按照慣例，最簡單的觀察方式就是直接觀察 `iptables` 的規則，這邊直接透過 `k8s-ngins-affinity` 這個關鍵字來查詢所有相關的規則

```bash=
$sudo iptables-save | grep k8s-nginx-affinity
-A KUBE-SEP-HDMJEKA4BFKBU6OK -s 10.244.0.145/32 -m comment --comment "default/k8s-nginx-affinity:" -j KUBE-MARK-MASQ
-A KUBE-SEP-HDMJEKA4BFKBU6OK -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.145:80
-A KUBE-SEP-Q5HAFBJX4HVXF6EM -s 10.244.0.144/32 -m comment --comment "default/k8s-nginx-affinity:" -j KUBE-MARK-MASQ
-A KUBE-SEP-Q5HAFBJX4HVXF6EM -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-Q5HAFBJX4HVXF6EM --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.144:80
-A KUBE-SEP-YFKOY7G33LWKGTLC -s 10.244.0.143/32 -m comment --comment "default/k8s-nginx-affinity:" -j KUBE-MARK-MASQ
-A KUBE-SEP-YFKOY7G33LWKGTLC -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-YFKOY7G33LWKGTLC --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.143:80
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.109.59.245/32 -p tcp -m comment --comment "default/k8s-nginx-affinity: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.109.59.245/32 -p tcp -m comment --comment "default/k8s-nginx-affinity: cluster IP" -m tcp --dport 80 -j KUBE-SVC-UBXGHWUUHMMRNNE6
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-YFKOY7G33LWKGTLC --mask 255.255.255.255 --rsource -j KUBE-SEP-YFKOY7G33LWKGTLC
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-Q5HAFBJX4HVXF6EM --mask 255.255.255.255 --rsource -j KUBE-SEP-Q5HAFBJX4HVXF6EM
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -j KUBE-SEP-HDMJEKA4BFKBU6OK
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-YFKOY7G33LWKGTLC
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-Q5HAFBJX4HVXF6EM
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -j KUBE-SEP-HDMJEKA4BFKBU6OK
```

稍微看了一下可以發現規則數量變多了，每個 `Endpoints` 本身多出兩條的規則出來，所以此範例中因為有三個`Endpoints`，所以總共會多出六條新的規則。
```bash=
-A KUBE-SEP-HDMJEKA4BFKBU6OK -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.145:80
-A KUBE-SEP-Q5HAFBJX4HVXF6EM -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-Q5HAFBJX4HVXF6EM --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.144:80
-A KUBE-SEP-YFKOY7G33LWKGTLC -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-YFKOY7G33LWKGTLC --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.143:80
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-YFKOY7G33LWKGTLC --mask 255.255.255.255 --rsource -j KUBE-SEP-YFKOY7G33LWKGTLC
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-Q5HAFBJX4HVXF6EM --mask 255.255.255.255 --rsource -j KUBE-SEP-Q5HAFBJX4HVXF6EM
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -j KUBE-SEP-HDMJEKA4BFKBU6OK
```

在我們開始研究這些規則之前，我們還是要先來問自己一句話
**如果是我們自己來實作這個功能，我們會怎麼實作?**

假設需求就是 `ClientIP` ，相同來源`IP`地址所建立的新連線都要分配到相同的 `EndPoints` 來使用
直覺下，我們可以用類似 `Cache` 的概念來完成這個功能，其流程如下
1. 收到新的連線請求, 檢查該來源`IP`地址是否存在 `Cache` 中
2. 若存在，直接使用該 `Cache` 內關於的目標 `Endpoints` 來使用
3. 若不存在，則嘗試從 `EndPoints` 內挑選出一個目標，並且將結果記錄到 `Cache` 之中.

所以可以將該 `cache` 分成 `Read/Wrtie` 兩個功能面向來看待，以下圖來表示


![Imgur](https://i.imgur.com/r6Yr1eI.png)


上述的流程看起來滿直觀且合理的，但是這些流程在 `iptables` 的規則中到底要怎麼完成?

## recent modules

我們將前面6條新規則縮減到兩條來單獨觀察就好**(因為每個EndPoints會有兩條)**
```bash=
-A KUBE-SEP-HDMJEKA4BFKBU6OK -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.145:80

-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -j KUBE-SEP-HDMJEKA4BFKBU6OK
```

為了加深各位的印象並且能夠順利的解讀 `ClusterIP` 的原理，需要再次複習一下這張圖片，並且確保知道下圖中各個項目的含意。
![Imgur](https://i.imgur.com/9amwybH.png)


### Save
首先我們觀察第一條規則，其位於 `KUBE-SEP` 這個位置，這個其實就是真正執行 `DNAT` 的 `custom  chain`.
這邊做的事情與我們假想的流程完全一致, 當選出欲使用的 `Endoints` 並進行 `DNAT` 轉換之時，順便將該結果記錄到 `Cache` 內。
**若不存在，則嘗試從 `EndPoints` 內挑選出一個目標，並且將結果記錄到 `Cache` 之中.**

我們來仔細看一下這條規則
```bash=
-A KUBE-SEP-HDMJEKA4BFKBU6OK -p tcp -m comment --comment "default/k8s-nginx-affinity:" -m recent --set --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 10.244.0.145:80
```

裡面新增加的部份則是
```bash=
-m recent --set --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource
```

這邊我們要介紹一個新的 `iptables` 的擴充模組 **recent**. 但是礙於篇幅沒有辦法詳細介紹其所有用法以及原理。
我們可以將 `recent` 想成他提供一個簡單的類似 `key/value` 的 `cache` 機制，同時支援 **Read/Write** 等操作來存取該 `Cache`.

這邊就針對這參數進行一個簡單的介紹
1. -m recent: 使用擴充模組 `recent`
2. --set: 這次的行為想要進行儲存的動作，將某些 `key/value` 寫進到 `recent cache` 內
3. --name KUBE-SEP-XXXXXXXX: 這邊對應的就是存到 `cache` 內的 `Value`.
5. --mask 255.255.255.255: 這個搭配下一個參數使用
6. --rsource: 這邊代表是的我要用什麼當做 `key`, 這邊使用的是 `souruce` 就是所謂的封包來源`IP`地址,既然有`IP`地址，就可以搭配前面的`mask`來調整`IP`位址的範圍，這個範例中就是**/32**的設定，意味`IP`要完全一樣才行。

所以歸納一下，若今天已經選定了一個`Endpoints`要來使用，首先會先跳到屬於該 `Endpoints` 專屬於的 `custom chani` **KUBE-SEP-HDMJEKA4BFKBU6OK**.
在進行 `DNAT` 之前，會先透過 `recent cache` 的方式去紀錄下列的對映關係

**[來源IP地址] => KUBE-SEP-HDMJEKA4BFKBU6OK**

將上述的概念重新整理，目前的已知拼圖如下

![Imgur](https://i.imgur.com/p9Cff8e.png)


另外，之前有提到過 `iptables` 的每個指令都是`符合特定規則`，執行`特定行為`.
所以其實 `recent` 模組內關於 `Set/Write` 相關的操作永遠都是回傳`符合`，讓上層的規則可以繼續往下執行。
畢竟針對 `Set/Write` 這類型操作本身就沒有要比對任何東西，只是被拿來進行其他的操作而已。

### Read

看完了第一題規則後，接下來來看一下最後一條，其實也就是第二條規則

```bash=
-A KUBE-SVC-UBXGHWUUHMMRNNE6 -m comment --comment "default/k8s-nginx-affinity:" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-HDMJEKA4BFKBU6OK --mask 255.255.255.255 --rsource -j KUBE-SEP-HDMJEKA4BFKBU6OK
```

這條規則其實就是 `Cache` 裡面關於 `Read` 的操作，但是這邊有一個點要注意，因為 `iptables` 的規則就是一條一條根據`比對條件`來判斷要不要執行`特定行為`.
所以這邊沒有辦法用程式化的方式去從 `Cache` 裡面取得對應的 `EndPoints` 名稱。

我們先記住，該 `recent` 提供的方式是詢問請問該`Key`有沒有資料，有的話是不是這個`Value`。
在這種情況下，我們可以想像一下其運作原理。

針對每一條 `KUBE-SVC-XXX` 裡面的規則，依序每個 `EndPoints` 執行下列操作
1. 請問`Cachue`裡面有沒有 `來源IP位址` => `當前EndPoints` 的紀錄? 有的話就直接跳到對應的 `Endooints` 的`custom chain`去執行`DNAT`.
2. 如果沒有的話，嘗試第二個 `Endpoints`
3. 所有的 `Endpoints` 嘗試後都沒有結果，那就透過機率的方式選擇一個可用的 `Endpoints`

有了這些概念後，我們從參數的部分來直接看一下 `iptables` 實際上的下法
1. -m:recent 使用擴充模組 `recent`
2. --rcheck: 這邊我們執行 `READ` 的指令，要檢查 `cache` 內是否有對應的 `key/value`
5. --name: `value`, 就是 `Endpoints` 對應到的 `custom-chain name`
6. --rsouce/--mask: `key`, 封包的來源 `IP`
7. --seconds: 每個 `cache` 內的記錄都會有一個過期的時間，這個時間的意思是只有上次設定該 `ket/value` 的時間距離現在`N`秒內的才算數，已這個範例來說就是 `10800秒` 內的 `cache` 記錄才算數，如果是超過 `10800秒` 前記錄的，就當失效。
8. --reap: 這個是指每次查詢的時候，會將已經超過`有效時間` 的規則一併清除。

把這整個流程全部重新組合後，我們用下面的這張圖來描述關於整個 `SessionAffinity` 的概念與實作
`seconds` 相關的概念我就沒有加入到該圖片中了，因為篇幅有限，針對主要觀念去描述即可。

![Imgur](https://i.imgur.com/bKvZgw4.png)

# Summary
看到這邊，我們大概瞭解如何透過 `iptables` 的功能來達成 `SessionAffinity:ClientIP` 的功能，透過 `iptables` 的擴充模組 `recent` 提供類似 `key/value` 的 `cache` 機制來紀錄 `來源IP地址` 與 `Endpoints` 的關係。

如果對於 `iptables` 擴充模組有興趣的讀者，之後我會撰寫一些文章介紹 `iptables` 的架構以及如何自己撰寫一個 `iptables` 的擴充模組。

最後我們將本篇文章所學的概念與一直以來使用的關係圖給整合起來，當設定 `SessionAffinity` 時，我們會在`KUBE-SVC` 嘗試透過 `Recent/Cache` 的方式找到是否有使用過的 `Endpoints`
之後再真正執行 `DNAT` 的 `KUBE-SEP-XXXX` 時會不停的更新 `Recent/Cache` 內的資料以及時間，避免該筆資料過期。

![Imgur](https://i.imgur.com/NP5xsGg.png)

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