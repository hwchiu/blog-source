---
title: '[書本導讀]-GitOps後續'
keywords: 'GitOps,DevOps,CI/CD,Kubernetes'
tags:
  - Kubernetes
  - Devops
  - GitOps
description: >-
  本文為電子書本[GitOps: What You Need to Know
  Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now)
  的心得第六篇。已獲得作者授權同意
abbrlink: 24699
date: 2020-10-06 15:21:42
---

本文大部分內容主要擷取自 [GitOps: What You Need to Know Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now) ，已獲得作者授權同意

本文為 GitOps 系列文，主要探討 GitOps 的種種議題，從今生由來，說明介紹，工具使用到實作上的種種挑戰，讓大家可以從不同角度來學習 GitOps

# GitOps Alternatives

在考慮如何交付應用程式的概念時，除了 GitOps 之外，也是有其他的流派與概念可以使用。
如同前所述， GitOps 依賴於 Git 本身的架構，使用 Git 當作整個操作的資料庫，儲存以及管理應用程式的期望狀態。 相對於 GitOps 的其他流派則會有不同的做法，譬如可能會運行不同的軟體架構，該架構本身就會儲存所有跟環境有關的期望狀態，然後該軟體則會想辦法將這個擁有全部期望狀態的應用程式給構建出來。

以下是相關的專案
1. [Humanitec](https://humanitec.com/)
2. [Harness](https://harness.io/)
3. [Kong](https://konghq.com/)
4. [Spinnaker](https://spinnaker.io/)

上述這些專案在使用上情境都會比較限定，譬如 `Humanitec` 則是針對持續交付去使用的，本身則不是為了持續整合而設計。


# What's Next for GitOps?
直至現今， GitOps 鑑於開源軟體彼此的獨立性而達到很好的發展，作者也非常期待能夠有愈來愈多的產品，套件，甚至是整套的解決方案來整合這些開源軟體。譬如之前所提過的 `JenkinX` 就是一個採用各種不同開源軟體來處理不同面向問題，最後整合出一個完善的方案給使用者使用。

## Emergent Patterns
基於 GitOps 這種操作方式，也有一些新的想法被逐漸提出來探討如何將 GitOps 給套用真實世界的應用場景。一個常見的範例就是將不同環境的流水線給分開獨立，當今天有任何更動時，可以先於A環境進行測試，都沒有問題時則自動的套用到B環境。
一個實際的範例就是 kubestack 所推出的 [cluster_pair](https://www.kubestack.com/framework/documentation/cluster-pairs) 模型，可以讓一個 Repo 內同時管理多個 Cluster。

另外一個有趣的模式則是如何有效的將基礎架構(infrastructure)與應用程式(applications)所使用到的狀態檔案分開成不同的 Git repo。 這兩個 repo 本身的層級不同，存取權限也不同，設計與實作上則會有諸多考量要處理，同時也要確保任何新的應用程式都可以順利部署不會出錯。

作者認為目前上述的一些模式都還比較屬於早期階段，更多都是手動創建與處理，期望有一天能夠有愈來愈多的解決方案將這些模式給實作並整合近來。

# Conclustion
接下來針對這六篇文章進行個總結

GitOps 這個概念起源自 2017 年，此後就一直有者良好的發展

GitOps 本身圍繞者三個主要核心概念
1. 透過單一控管的方式來維護歷史紀錄與操作
2. 透過宣告式的方法來定義資源的期望狀態
3. 透過 Control Loop 的方式來同步資源狀態

目前市面上有一些軟體非常適合組合出 GitOps 的解決方案
1. Git
2. Terraform
3. Kubernetes
4. Flux

直到現在，有愈來愈多的組織與廠商開始嘗試 GitOps 這樣的操作，雖然其工具與產品都還在處於非常早期階段，我們還是可以未來期待會有愈來愈多的選擇。

接下來我們則是可以期待去看看 GitOps 是否可以在這個市場上存活，畢竟 GitOps 嘗試解決許多過去的部署問題，譬如
1. Control: 
Zero-touch 的系統讓你能夠去限制誰能夠對你的系統做什麼
2. Audit:
版本控制加上宣告式的格式可以讓你知道是誰做的，做了什麼，什麼時候做的
3. Inventory:
宣告式的做法可以讓你確保你的系統於任意時間點應該要長怎樣

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



