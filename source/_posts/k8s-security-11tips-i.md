---
title: 11個保護你 Kubernetes 集群的技巧與觀念(上)
keywords: 'kubernetes,security,Linux,k8s,TLS'
tags:
  - Kubernetes
  - Security
  - Linux
abbrlink: 27871
date: 2018-07-24 18:04:31
description:
---

本篇文章的原文為 [11 Ways Not to Get Hacked](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/#11-run-a-service-mesh)

本文作者將這 11 個技巧與觀念分成三大部分來
分別是
1. The Control Plane
2. Workloads
3. The Future

每個部分除了文章內容的敘述外，其實也都補充了不少相關的資源與補充。
所以接下來我會使用兩篇文章的方式來記錄這 11 個觀念/技巧並且加上我個人的看法

本篇文章後續 [11個保護你 Kubernetes 集群的技巧與觀念(下)](https://www.hwchiu.com/k8s-security-11tips-ii.html#more)

<!--more-->

## The Control Plane
`The Control Plane` 所代表的就是 `Kubernetes` 的控制面，其擁有的權力跟能力非常的巨大，從最基本資源的創建與監控 (Pod/Deployment/Service)，資源的調度(Pod Schedule) 以及包括所有 Kubernetes 上資源的存取 (Secret/ConfigMap)。

由於 `The Control Plane` 這邊能做的事情實在太多，一旦讓非系統管理者有機會接觸到這個部分，就有機會對整個集群造成意想不到不可挽回的事情。
因此第一個章節所要描述的就是如何安全的防護你的 `kubernetes`，讓任何惡意攻擊者沒有機會去操作你的 `kubernetes` 集群。

### TLS Everywhere
這邊的原則非常簡單，只要任何內部元件之間的溝通有支援 TLS, 就使用 TLS，沒有任何不用的理由
透過 [TLS](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A) 能夠確保這條連線受到保護，除了可以驗證 Server/Client 彼此的深分外，也能夠避免傳輸內容被竊聽。

事實上有提供 `TLS` 功能連線的元件也都有提供所謂的類 `Insecure` 方式連線，就是不依賴 `TLS` 來進行連線而是直接進行純文字傳送。
對於很多人來說，其實會覺得要設定 `Secure` 連線很麻煩，在 Server 端要產生準備好憑證，而且對於每個連線的 Client 端都要準備好對應的憑證，讓整個連線可以正常運行。
我自己是覺得除了進行研究方便快速測試之外，其他的情境應該都要盡可能的使用 `Secure` 連線來確保連線安全。

那在 `Kubernetes` 中，到底有多少個元件之間有 `TLS` 連線的存在?

根據 [Lucas Käldström](https://docs.google.com/presentation/d/1Gp-2blk5WExI_QR59EUZdwfO2BWLJqa626mK2ej-huo/edit#slide=id.g1e639c415b_0_56) 於 [kubeadm Cluster Creation Internals: From Self-Hosting to Upgradability and HA](https://docs.google.com/presentation/d/1Gp-2blk5WExI_QR59EUZdwfO2BWLJqa626mK2ej-huo/edit#slide=id.g1e639c415b_0_56) 所描繪的流程圖，我們可以清楚地看到在 `kubernetes` 元件彼此之間的溝通，除了 `Kubernetes` 本身元件之外，也包含了第三方的插件的開發。

![](https://i.imgur.com/eovKE0P.png)

#### Master Node
首先，`Master` 上面的元件可以分成兩種交流方式，分別是 `gRPC` 以及 `Protobuf`
這邊我認為是因為 `etcd` 本身不是屬於 `kubernetes` 自行開發的元件，所以在與之溝通上就會必須依賴本身已經存在的格式與規範。
由於 [gRPC](https://grpc.io/) 本身是基於 `HTTP2` 的方式來傳輸封包，所以透過 `TLS` 加密是完全沒有問題的

而上圖中的 `Controller Manager`, `API Server` 以及 `Scheduler` 這些 `Kubernetes` 自行開發的組件彼此之間透過 `Protobuf` 來規範這些格式，都可以在這邊 [design-proposals](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md) 看到開發與設計的規範。
目前上述三個元件互相溝通也是支援 `TLS` 連線的，如果夠熟悉 `kubernetes` 手把手建置過程的讀者就知道整個集群內有非常多的 key/cert 等資訊要設定與配置，非常的複雜。

#### Cross Nodes

在 `kubernetes` 的架構下，除了所謂的 `master` 節點外，還有所謂的 `minion` 節點，而 `minion` 則會透過機器上的 `kubelet` 與 `API Server` 進行溝通。
如同上述的 `Protobuf` 的走法，這邊也有公開的設計規範以及 `API`，最後溝通的部分也支援 `TLS` 加密。

圖片中還有提到 `CNI`,`CRI`, `OCI` 等這部分則是容器相關資源標準化介面，從`Network`,`RunTime`等不同標準，實際上使用者會使用哪些都是可以自行抽換的。
此外，不少網路的 `CNI` 也都有透過 `etcd` 的方式去存取資料，這意味者對於每個 `CNI` 也都必須要幫忙準備相關的 `etcd` 憑證來建立 `TLS` 連線，千萬不可以為了方便就捨棄安全性。


#### TLS 帶來的問題
`TLS` 聽起來很美好，實際上在部屬的時候卻會帶來不少困難，最明顯的就是動態增加節點需要額外的心力去處理憑證相關的問題。
為了解決這個問題，可以參考這篇由 [Todd Rosner
](https://medium.com/@toddrosner) 撰寫的這篇文章 [Kubernetes TLS bootstrapping](https://medium.com/@toddrosner/kubernetes-tls-bootstrapping-cf203776abc7), 裡面有詳細介紹 `Kubernetes TLS bootstrapiing` 的原理與設計以及如何用來解決動態擴充節點的問題，非常值得一讀。

### Enable RBAC with Least Privilege, Disable ABAC, and Monitor Logs

`Kubernetes` 在 `Authorization` 方面也有許多的安全問題需要注意，想要瞭解全文可以直接參考官網文件 [Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)

簡單來說，目前 `kubernetes` 支援的 `Authorization` 總共有四種方法，分別是
1. Node
2. RBAC
3. ABAC
4. Webhook

本篇文章不會探討這四種的差異及比較，之後有時間會再寫文章來介紹這幾種並且分享看法。
這邊要知道的就是請盡量使用 `RBAC` 並且在設定權限的時候，不要把權限全開，請根據用到的權限開啟特定的權限來達到最大的保護。

如果對於 `RBAC` 想要詳細暸解，可以參考這篇文章 [RBAC Support in Kubernetes](https://kubernetes.io/blog/2017/04/rbac-support-in-kubernetes/)

有寫過 `RBAC` 的讀者就會知道，其實寫 `RBAC` 非常的麻煩，必須要不停的 `try and error` 去找出到底自己的應用程式用到了哪些權限，一個一個的補上去。
這過程非常疲倦但是為了最高的安全性，還是要請大家準確地執行。

不過為了解決這個問題，這邊特別介紹一個工具 [audit2rbac](https://github.com/liggitt/audit2rbac)
只要你的 `kuberntes` 是 `v.10.0` 之後，就可以開啟一個 `beta` 的功能 `audit log`.
而上述的工具可以幫你解析這些 `audit log` 來判斷你的權限還缺少哪些，然後幫你產生對應的 `RBAC` 檔案，聽起來滿好用的，這部分還需要找時間來實際玩玩看，看看是否真的如敘述般的好用。
當然有興趣的讀者也可以先行嚐鮮來使用看看這個工具

### Use Third Party Auth for API Server
原文認為透過一個類似 `SSO(Single Sign On)` 這種集中的方式能夠有效的去控管使用者的權限，特別是當使用者有任何調度更動時。

對於 `Kubernetes` 來說，如果整合第三方服務的認證，譬如 `Google/GitHub`，就可以在有大量使用者有任何異動之時不需要一直重新調整設定 `Kubernetes API server` 

除了上述服務外，作者還介紹了 [OIDC(OpenID Connect Identity)](https://github.com/micahhausler/k8s-oidc-helper) 與 `kubernetes` 的一些使用情境，可以在這個專案 [k8s ODIC helper](https://github.com/micahhausler/k8s-oidc-helper) 這邊看到使用方法。

這邊我本身也還沒有很熟悉，需要花更多時間來理解這邊的觀念到時候再跟大家介紹這些應用與概念。

### Separate and Firewall your etcd Cluster
原文作者說道, `etcd` 在 `kubernetes` 集群內扮演一個非常重要也是非常關鍵的角色，其儲存了所有 `kubernetes` 集群內的所有資訊，包含了各式各樣的設定以及 `kubernetres` 的 `secrets` 資源。
為了妥善保護好 `etcd`，作者認為 `etcd` 本身在安全防護上要跟 `kubernetes`集群是分開處理的。

對於 `etcd` 這樣的角色來說，只要讓攻擊者有`寫入`的能力，就意味者攻擊可以扮演一個 `root` 的角色來操控整個 `kubernetes` 集群。
若是攻擊者只有`讀取`的能力，也是有可能會讓攻擊者透過 `etcd` 取得道各式各樣機密的資訊。

這邊舉一個範例來說明 `etcd` 被攻擊會有多大的影響。
1. `kubernetes scheduler` 會透過 `etcd` 來找尋還沒有運行起來的 `pod`, 然後尋找一個可行的 `pod` 來運行。
2. 如果今天有使用者想要創立一個 `pod`, 則該 `pod` 會先在 `API server` 端進行驗證，確認一切參數都合法且符合條件後，就會把該 `pod` 的資訊寫入到 `etcd`.

綜合以上兩點，如果今天有惡意使用者直接透過 `etcd` 去修改 `pod` 本身的屬性，譬如`PodSecurityPolicies` 這方面跟安全性/權限有關的任何設定，就可以直接對這個 `pod` 產生非預期的結果，這邊就會有很大的安全性漏洞。

要怎麼保護 `etcd`?
作者認為
1. 一定要上 TLS 加密
2. etcd 應該要部署在特定的節點上
3. 為了減少私鑰被偷取並且在一般的 `worker` 節點上使用，管理者可以針對 `etcd` 集群與 `API server` 進行防火牆的設定，只有 `API server` 才能夠跟 `etcd` 集群連線，避免任何 `worker` 節點上有任何機會去存取 `etcd`

### Rotate Encryption Keys
對於 `Security` 來說，定期的轉換你使用的金鑰與憑證能夠降低問題發生時的最大風險程度。
舉例來說，若今天不小心金鑰遺失了，若系統上面沒有去定期改變，則攻擊者可以無時無刻的都對系統進行非預期的操作，然而若金鑰有定期去改變，則攻擊者在一定時間後就沒有辦法繼續對集群操作了。

對 `kubernetes` 來說，針對 `kubelet` 這隻應用程式來說，目前有提供這樣的功能，能夠定期的去轉換，這邊可以參考官方文件 [Certificate Rotation
](https://kubernetes.io/docs/tasks/tls/certificate-rotation/#enabling-client-certificate-rotation)
但是對於採用`對稱性加密`的 `API Server` 來說則是沒有提供這個能力，這邊我暫時還不清楚差異性，對於安全性沒有這麼熟悉。
原文則表示這部分需要手動的去定時轉換來提高安全性，下列這篇文章有介紹該如何操作 [Kubernetes Secrets Encryption
](https://www.twistlock.com/2017/08/02/kubernetes-secrets-encryption/)，並且提到若你採用的集群是 `GKE/AKS` 這種雲端服務商提供的 `Kubernetes` 平台，則背後都會自己幫你進行這方面的保護。



## Summary

其實原文只有短短的敘述過去，但是其實每篇內容都有非常詳細的資訊可以繼續擴展，能的話建議讀者花點時間把文章內的連結都讀了一遍，我相信會對 `kubernetes` 內部的各種機制有更深層的瞭解，可以學到更多的東西。


## 來源參考
- [11 Ways (Not) to Get Hacked](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/)
- [kubeadm Cluster Creation Internals: From Self-Hosting to Upgradability and HA
](https://docs.google.com/presentation/d/1Gp-2blk5WExI_QR59EUZdwfO2BWLJqa626mK2ej-huo/edit#slide=id.g1e639c415b_0_56)
- [Kubernetes TLS bootstrapping
](https://medium.com/@toddrosner/kubernetes-tls-bootstrapping-cf203776abc7)
