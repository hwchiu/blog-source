---
title: '[Kubernetes] How to Implement Kubernetes Service - NodePort'
keywords: 'Kubernetes,Service,Linux,k8s,iptables,NodePort'
abbrlink: 56589
date: 2018-08-25 01:17:08
tags:
  - Kubernetes
  - Linux
  - Iptables
description: 在前述中我們已經學過了什麼是 kubernetes service 以及如何實現 ClusterIP 這種類型的 service. 透過對 iptables 的探討與研究, 我們可以理解到 ClusterIP 本身會提供一個虛擬的 IP 地址,接下來只要跟這個地址有關的封包,都會透過 DNAT 的方式進行轉換找到最後的 Endpoint IP. 至於如何選擇的部分,則是透過機率的方式去尋找. 接下來我們要來探討另外一個也是很常使用的 kubernetes service 類型, 也就是 NodePort. NodePort 本身包含了 ClusterIP 的所有功能, 此外也額外開啟了一個新的存取方式. 透過直接存取節點的 IP 地址配上一個設定好的 Port, 也可以將該封包直接送到最後面的容器應用程式. 因此本文也要延續上一篇的思路,繼續研究 iptables 的規則來探討 NodePort 到底是如何實現的

---

# Preface
本文章是屬於 `kubernetes` service 系列文之一，該系列文希望能夠與大家討論下列兩個觀念
1. 什麼是 `Kubernetes Service`, 為什麼我們需要它？ 它能夠幫忙解決什麼問題
2. `Kubernetes Service` 是怎麼實現的?， 讓我們用 iptables 來徹徹底底的理解他

相關文章:
[[Kubernetes] What is Service](https://www.hwchiu.com/kubernetes-service-i.html)
[[Kubernetes] How To Implement Kubernetes Service - ClusterIP](https://www.hwchiu.com/kubernetes-service-ii.html)
[[Kubernetes] How To Implement Kubernetes Service - SessionAffinity](https://www.hwchiu.com/kubernetes-service-iiii.html)


本文銜接上篇文章，繼續透過對 `iptables` 的分析來研究 `kubernetes service` 中 `NodePort` 的實作原理。

<!--more-->
`NodePort` 的功能就如同字面上的意思一樣,`Node Port`, 提供了一種透過存取叢集節點上事先定義好的`Port Number` 就可以輾轉存取到後端的真正服務。

作為一個靠腦力生存的人，每次遇到全新概念的時候，都要問問自己幾個問題
1. 這個概念是想要解決什麼問題?
2. 什麼時候會用到?
3. 如果是我，我會怎麼實作?

# Why We Need NortPort
`NodePort` 本身是屬於 `kubernetes service`的一環，自然就是要提供一個方式可以讓外部來存取集群內的服務，而且可以不用去理會後面這些容器們的真實`IP`地址。

既然已經有前面的 `ClusterIP` 提供了一種叢集內存取的方式，什麼情況下我們會需要 `NodePort` 這種透過存取節點的方式?

這邊使用一個下列的範例來解釋可能的情況

以下只是一種範例，但是未必是最佳解

假設今天有一個試驗環境，在`Cloud Provider(Google/Azure/AWS...etc)`中架設了一個`kubernetes`叢集，裡面透過 `nginx` 的方式部屬了一個網頁伺服器。
與此同時，我希望該叢集能夠提供下列的特性供我使用
- 我希望管理人員可以不需要去擔心該 `nginx` 的狀態，其網頁服務能夠一直正常運作。
- 我可以在任意地方直接連接到該 `nginx` 提供的網頁伺服器服務


為了滿足第一個條件，我們可以透過 `kubernetes deployment` 的方式去確保 `nginx` 的容器處於一種運行的狀態。
為了滿足第二個條件，我們可以透過 `kubernetes service` 的方式去連接上述的 `nginx` 容器們並且提供一種接口讓外部存取
在這種情況下，只要`kubernetes`叢集內的節點擁有一個固定的對外`IP`地址，同時 `kubernetes server` 透過 `NodePorts` 的方式提供該`nginx`往外存取的能力。
這種情況下，我們就可以在任何地方，透過直接存取該節點的對外`IP`地址，然後間接透過`NodePort`的功能存取到集群內的`nginx`服務。


# How It Works
已經有了前述關於 `ClusterIP` 運作的概念後，其實要探討 `NodePort` 是如何實現的就非常簡單了。

我們先快速複習一下 `ClusterIP` 的運作流程
![Imgur](https://i.imgur.com/xoPvipq.png)
1. 封包若來自叢集上的應用程式/節點，則跳到 `KUBE-SVC
2. 如果封包的目標`IP`地址是 `ClusterIP` 所提供的虛擬`IP`地址, 則跳到 `KUBE-SVC-XXXX`
3. `KUBE-SVC-XXX` 裡面根據機率的方式，選到一個 `Endpoints`，最後跳到 `KUBE-SEP-XXX`
4. `KUBE-SEP-XXX` 裡面執行 `DNAT`, 將封包的目標地址改成真正的容器地址，然後轉發

有了上述的概念，我們如果要支援 `NodePort` 這種能夠透過`節點IP`的方式存取的話。
想了一下，其實就是把上述的(1)/(2)改掉就好，能夠跳到 `KUBE-SVC-XXX`的話，後續就完全一致了。

接下來，我們繼續使用[kubeDemo](https://github.com/hwchiu/kubeDemo)來進行相關的服務部屬以及`iptables`規則研究。

首先，我們先在環境內部署相關的 `nginx` 以及 `kubernetes service(NodePort)`

```bash=
vortex-dev:06:36:12 [~/go/src/github.com/hwchiu/kubeDemo/services](master)vagrant
$kubectl apply -f deployment/nginx.yml
deployment.apps/k8s-nginx created

vortex-dev:06:36:18 [~/go/src/github.com/hwchiu/kubeDemo/services](master)vagrant
$kubectl apply -f service/nginx-node.yml
service/k8s-nginx-node created
```

這邊就不再敘述太多跟 `service/endpoints` 相關的資訊與位置，直接從 `iptables` 的角度出發。

我個人非常喜歡 `kubernetes`的一點就是所有的 `iptables` 的規則都會下註解，所以其實可以很輕易的透過 `grep` 的方式找到相關的規則。
以上述的範例來說，我們在 `default` namespace 中部屬了一個 `k8s-nginx-node` 的 `kubernetes service`.
所以透過 `grep default/k8s-nginx-node` 的方式就可以過濾出所有跟這個`Service`有關的所有規則。


```bash=
vortex-dev:03:17:35 [~]vagrant
$sudo iptables-save  | grep default/k8s-nginx-node
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp --dport 30136 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp --dport 30136 -j KUBE-SVC-RD5DSC6PXE26GCYZ
-A KUBE-SEP-VRKO3GZ2XUCPVWY5 -s 10.244.0.115/32 -m comment --comment "default/k8s-nginx-node:" -j KUBE-MARK-MASQ
-A KUBE-SEP-VRKO3GZ2XUCPVWY5 -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp -j DNAT --to-destination 10.244.0.115:80
-A KUBE-SEP-YNJKNN6SS5424R7C -s 10.244.0.113/32 -m comment --comment "default/k8s-nginx-node:" -j KUBE-MARK-MASQ
-A KUBE-SEP-YNJKNN6SS5424R7C -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp -j DNAT --to-destination 10.244.0.113:80
-A KUBE-SEP-ZGMDZ7UNNV74OV5B -s 10.244.0.114/32 -m comment --comment "default/k8s-nginx-node:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ZGMDZ7UNNV74OV5B -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp -j DNAT --to-destination 10.244.0.114:80
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.98.128.179/32 -p tcp -m comment --comment "default/k8s-nginx-node: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.98.128.179/32 -p tcp -m comment --comment "default/k8s-nginx-node: cluster IP" -m tcp --dport 80 -j KUBE-SVC-RD5DSC6PXE26GCYZ
-A KUBE-SVC-RD5DSC6PXE26GCYZ -m comment --comment "default/k8s-nginx-node:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-YNJKNN6SS5424R7C
-A KUBE-SVC-RD5DSC6PXE26GCYZ -m comment --comment "default/k8s-nginx-node:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-ZGMDZ7UNNV74OV5B
-A KUBE-SVC-RD5DSC6PXE26GCYZ -m comment --comment "default/k8s-nginx-node:" -j KUBE-SEP-VRKO3GZ2XUCPVWY5
```

我們快速的掃過所有的規則，根據 `custom chain` 來看，分成四個部分
1. KUBE-NODEPORTS
2. KUBE-SEP-XXXX
3. KUBE-SERVICES
4. KUBE-SVC-XXXX

這邊目前只有 `KUBE-NODEPORTS` 還沒有看過，剩下的都跟 `ClusterIP` 是一樣的功能的。

NodePort 的功能基於 ClusterIP 之上再添加新功能，所以本來 Cluster 該有的規則對於 NodePort 來說都不會少

# KUBE-NODEPORTS
我們仔細觀察 `KUBE-NODEPORTS` 相關的兩條規則
```bash=
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp --dport 30136 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/k8s-nginx-node:" -m tcp --dport 30136 -j KUBE-SVC-RD5DSC6PXE26GCYZ
```

第一條規則的目標是 `-j KUBE-MARK-MASQ`, 這部份是跟 `SNAT` 有關的，這個之後有機會再寫額外的文章來介紹 `SNAT` 相關的功能以及處理方式。
這邊只要知道這是修改封包的來源`IP`位址即可
第二條規則比較重要，我們可以觀察到
1. 其比對條件是 `-m tcp --dport 30136`.
2. 符合條件後執行的行為是 `-j KUBE-SVC-RD5DSC6PXE26GCYZ`

```bash=
vortex-dev:03:34:14 [~]vagrant
$kubectl get svc
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
k8s-nginx-node   NodePort    10.98.128.179   <none>        80:30136/TCP   1d
```

根據查詢 `kubernetes svc` 的結果，我們可以觀察到透過存取 `30136/TCP` 的方式就可以存取 `NodePOrt`.
而這個資訊與我們前面看到的 `KUBE-NODEPORTS` 這邊的規則完全一樣
最後可以發現當規則一致時，就會跳到 `KUBE-SVC-XXX` 去進行 `endpoints` 的挑選以及相關的 `DNAT` 功能。

哪接下來的問題只剩下一個
到底封包什麼時候會進入到 `KUBE-NODEPORTS` ? 只要釐清這個問題，剩下的處理方式就都跟 `ClusterIP` 完全一樣了。
這時候我們就要一條一條 `iptables` 的規則來慢慢查詢

我偷懶直接使用 `-j KUBE-NODEPORTS` 的方式來查詢，到底誰會跳入 `KUBE-NODEPORT`
```bash=
vortex-dev:03:43:42 [~]vagrant
$sudo iptables-save  | grep "\-j KUBE-NODEPORTS"
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

這個規則非常有趣，首先我們可以觀察到，他在 `KUBE-SERVICES` 這個 `custom chain` 裡面。 接下來可以觀察他的註解
```
kubernetes service nodeports; NOTE: this must be the last rule in this chain
```
然後看一下比對條件以及執行目標
1. -m addrtype --dst-type LOCAL
2. -j KUBE-NODEPORTS

第一個比對條件我們從文字上來解讀，只要封包的目標`IP`地址是屬於本節點上的任何網卡`IP`。
只要符合上述規則，就會跳到 `KUBE-NODEPORT` 裡面進行比對，然後就按照前述的去處理了。

對於 --dst-type LOCAL 有興趣的人可以嘗試閱讀下列這個檔案
https://elixir.bootlin.com/linux/v4.7.8/source/net/netfilter/xt_addrtype.c#L119
https://elixir.bootlin.com/linux/v4.7.8/source/include/uapi/linux/rtnetlink.h#L203
看看 kernel 內大致上是怎麼處理這系列操作的

到這邊我們整理一下所有的思路。
1. NodePort 也是倚賴 `KUBE-SERVICES`，當封包目標是本地端的 `IP` 位置的時候，就會跳到 `KUBE-NODEPORT` 裡面去比對 `protocol/port` 來進行後續跟 `ClusterIP` 相同的處理
2. 所有的 `kubernetes NodePort service` 都會共用同一個 `KUBE-NODEPORT`, 因此所有的 `NodePort` 使用的 `Port` 都不能一樣

我們用下列這張圖來總結 `NodePort` 的運作

![Imgur](https://i.imgur.com/9amwybH.png)


# PortBinding
由於 `NodePort` 會使用到節點上面的 `Port` 來提供服務
但從 `iptables` 的規則觀察下，其實 `NodePort` 所用到的 `Port` 就是一個虛擬的 `Port`，譬如上述範例的 `30136`。
為了避免有任何應用程式之後將 `NodePort` 要用到的 `Port` 給拿去使用，導致整個有任何非預期的行為出現
- 譬如某服務想要用 30136 port, 但是所有的封包都被 iptables 導走了，導致該服務一直沒有辦法接收到真正的連線

為了解決這個問題就是不要讓任何應用程式有機會使用到 `30136` 的連接埠，因此每個節點上面的 `kube-proxy` 就會幫忙做這件事情。

一旦 `NodePort` 成功建立後，就會將該 `Port` 給使用走，讓其他的應用程式沒有機會使用。

這部份我們可以透過 `netstat` 的指令來觀察
```bash=
vortex-dev:04:08:18 [~]vagrant
$sudo netstat -ltpn | grep 30136
tcp6       0      0 :::30136                :::*                    LISTEN      10181/kube-proxy
```

這點跟 `docker` 的想法是很類似的，不過 `docker` 所啟用的 `docker-proxy` 其實也會幫忙 `forward` 這些封包，而不是單純的搶占避免服務失效而已。
```bash=
vortex-dev:01:08:39 [~/go/src/github.com/linkernetworks/vortex/vendor](hwchiu/VX-62)vagrant
$sudo docker run -d -p 5566:80 nginx
f4b6b72ad82c170a92cd6ea272fc8d665b69835b8508d20e1ac2b220b2ba5b31
vortex-dev:01:08:43 [~/go/src/github.com/linkernetworks/vortex/vendor](hwchiu/VX-62)vagrant
$ps axuw  | grep docker-p
root     21499  0.0  0.0  59068  2852 ?        Sl   01:08   0:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 5566 -container-ip 172.18.0.2 -container-port 80
```

# Summary
本章節我們仔細的討論了 `NodePort` 各種面向的概念，最後發現其實 `NodePort` 的規則非常簡單，建立於 `ClusterIP` 之上。
只要能夠掌握 `ClusterIP` 是如何運作的，回過頭來看 `NodePort` 就不難理解這整個過程。

最後繼續使用這張圖作為總結，希望大家這時候都能夠順利的看懂這張圖要表達的一切概念
![Imgur](https://i.imgur.com/9amwybH.png)

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
