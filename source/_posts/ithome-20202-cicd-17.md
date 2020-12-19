---
title: 鐵人賽系列文章- Day 17 - GitOps 與 Kubernetes 的整合
keywords: 'Kubernetes,DevOps,Linux,K8s'
tags:
  - ITHOME
  - DevOps
  - Kubernetes
description: ITHOME-2020 系列文章
abbrlink: 25928
date: 2020-12-19 12:36:13
---

上篇文章我們探討了 GitOps 的概念，但是概念歸概念，實作歸實作，有時候實作出來的結果跟概念不會完全一致，因此最後的使用方式與優缺點還是要看實作的細節。

今天我們就來看看 GItOps 這個概念要怎麼與 Kubernetes 整合。

首先，前述 GitOps 的概念中，我們提到一個代理人程式，這個程式要能夠管理左看 Git Repo, 右看 Kubernetes ，那由於這個程式本身要能夠有能力去讀取 Kubernetes 內的資源狀態，同時也要有能力對其修改，勢必要獲得一些操控權限。

設想一個情境，如果今天這個代理人程式其座落於 Kubernetes 外，我們終究還是要為他準備一份 KUBECONFIG，這樣其實也是會增加安全性的風險，但是如果把這個程式放到 Kubernetes 裡面，讓其擁有存取 Kubernetes 能力的部分就相對好解決，這樣可以減少一些安全性的風險。



# 程式碼架構

GitOps 的架構下，因為都會把資源的狀態檔案都放在 Git，所以這時候就會有一些不同的做法，舉例來說

1. 應用程式原始碼以及相關的 Yaml 放一起
2. 應用程式原始碼以及相關的 Yaml 分開到不同的 Repo 去放

GitOps 的原則我認為採用 (2) 是比較好實現的，因為我們可以很明確地將應用程式與部署資源給分開，這兩個 Repo 所維護跟控管的團隊也有所不同。同時 Yaml Repo 內的所有更動跟只會跟部署資源的狀態有關，這樣對於維護，追蹤任何變動，甚至要退版等需求都比較好實現。

如果將程式碼跟相關 Yaml 放在同樣一個 Repo 內，那想要針對部署狀況進行退版，就有可能也會導致程式碼本身功能也一併被退版，這就不是期望的結果。

不過這邊所提的都只是一些各自的特性與優缺點，沒有一個絕對的解決方案跟絕對的正確與否，還是要根據 GitOps 的實作方式以及團隊的習慣方式去選擇。

接下來的架構圖都會基於 (2) 的方式去介紹



# 架構一

我們用下面的架構圖來看一下一個 GitOps 與 Kubernetes 整合範例一

這個架構下，我們會在 Kubernetes 內安裝一個 Controller，也就是前文所提的代理人程式，這個程式本身要有辦法與遠方的 Git(Yaml) Repo 溝通。

接下來其運作流程可能是

1. 開發者對 Git(Application) Repo 產生修改，這份修改觸發了相關的 CI Pipeline ，這過程中經歷過測試等階段，最後將相關的 Image 給推到遠方的 Container Registry。
2. 系統管理者針對 Git(Yaml) Repo 產生修改，這份修改觸發了相關的 CI Pipeline, 這過程中會針對 Yaml 本身的格式與內容進行測試，確保沒有任何出錯。
3. 當 Git(Yaml) Repo 通過 CI Pipeline 而合併程式碼後，接下來 Kubernetes 內的 Controller 會知道 Git(Yaml) Repo 有更新
   1. 一種是 Git 那邊透過 Webhook 的方式告訴 Controller
   2. 一種是 Controller 定期輪詢後得到這個結果
4. 同步 Git(Yaml) Repo 裡面的狀態描述檔案與 Kubernetes 叢集內的狀態，確保目前運行狀態與 Git 內的檔案描述一致
5. 如果今天不想要 (3) 這個步驟的自動化，也可以由管理員經過確認後，手動要求 Controller 去同步 Git 並更新





![](https://i.imgur.com/KbYGBqd.jpg)

上述的架構聽起來運作起來都滿順暢，但是對於開發者的叢集來說(假設開發者會有一個遠方的 Kubernetes 用來測試功能)可能會不便利，只要這些更新非常頻繁，那就意味者要一直不停的去修改 Git Yaml Repo 的內容，雖然一切都按照概念在運行，但是操作起來可能會覺得效率不一定更好。

因此也會有人將上述的一些架構整合，譬如當 Image 推向到遠方 Container Registry 後，利用程式化的功能於 CI Pipeline 的最後階段自動的對 `Git(Yaml) Code Changed` 去送一筆 Commit 並自動觸發後面的 Controller 同步行為。

GitOps 的概念很活，很多種做法都可以，並沒有要求一定要怎麼實作才可以稱為 GitOps，最重要的是你們的工作流程是否可以達到如同 GitOps 所宣稱的效果，就算不走 GitOps，只要可以增加開發跟部署的效率，減少問題就是一個適合的架構。



# 架構二

接下來我們來看一下另外一種參考架構，這種架構希望可以解決 Contaienr Image 頻繁更新的問題，因此 Controller 本身又會多了監聽 Container Registry 的能力。

當 Controller 發現有新的版本的時候，只要這個版本號碼有符合規則，就會把新的版本資訊給套用到 Kubernetes 裡面。

但是因為 GitOps 的原則是希望以 Git 作為單一檔案來源，如果這樣做就會破壞這個規則，因此這時候 Controller 就要根據當前 Image Tag 的變化，把變化內容給寫回到 Git(Yaml) Repo 之中。

這也是為什麼下圖中 Controller 要對 Git(Yaml) Repo 進行更新與撰寫新 Commit 的原因。

也因為這個原因，我們的 Controller 也必須要對該 Git(Yaml) Repo 擁有讀寫的能力，這方面對於系統又會增加一些設定要處理



![](https://i.imgur.com/baa65WB.jpg)



以上兩種架構並不是互斥，是可以同時存在的，功能面方面就讓各位自己取捨，哪種功能我們需要，帶來的利弊誰比較大。

下一篇我們將帶來其中一個開源專案 ArgoCD 的介紹，看看 ArgoCD 如何實踐 GitOps 的原則


# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4N7G

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
