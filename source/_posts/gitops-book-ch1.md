---
title: '[書本導讀]-GitOps 到底解決了什麼問題'
keywords: 'GitOps,DevOps,CI/CD,Kubernetes'
tags:
  - Kubernetes
  - Devops
  - GitOps
description: >-
  本文為電子書本[GitOps: What You Need to Know
  Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now)
  的心得第一篇。已獲得作者授權同意
abbrlink: 44855
date: 2020-09-21 20:34:41
---



本文大部分內容主要擷取自 [GitOps: What You Need to Know Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now) ，已獲得作者授權同意

本文為 GitOps 系列文，主要探討 GitOps 的種種議題，從今生由來，說明介紹，工具使用到實作上的種種挑戰，讓大家可以從不同角度來學習 GitOps。


# 前言
GitOps 這個新穎的部署概念近年來於網路上的討論有愈來愈多趨勢，而本篇系列文主要會針對  [GitOps: What You Need to Know Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now)  這本電子書來進行學習，來跟大家一起分享 GitOps 的前世今生，以及未來走向

# 前世
GitOps 這個概念是來自一篇於 2017 年由 `Weaveworks` 所發表的文章 [GitOps - Operation by Pull Request](https://www.weave.works/blog/gitops-operations-by-pull-request) 。GitOps 這個詞是由 Git + Operations 一起組合的，跟 DevOps 有異曲同工之妙，都在探討跨部門合作下的軟體開發與部署流程。 GitOps 更希望能夠透過 Git 的特性來減少 Operations 團隊的負擔，讓整個軟體的維護與部署更佳流暢。

# GitOps 想解決的問題
首先，我們先來探討 GitOps 到底想要解決什麼問題，為了更好的呈現這個概念，我們使用 `anti-patterns` 這種想法來闡述目前遇到的問題，最後探討 GitOps 能夠如何介入來解決。

## Deplyoment By Hand
軟體開發團隊現在都非常習慣使用版本控制等工具來開發與測試軟體，同時也會建置相關的部署產物。對於開發者來說，手動部署這些應用程式到目標環境中並不是一個不常見的選項。

雖然大部分的情況下，都可以透過撰寫一些腳本來盡可能的自動化這些部署行為，但是開發團隊跟維運團隊兩者大多情況下還是會有交集，需要合作來處理問題，特別是一些特定的環境，譬如整合環境，正式生產環境等。而這些合作可能還是會有一些手動操作介入。

## State In A Spreadsheet
維運團隊總是會希望能夠透過一些權限控管即可分享的系統來追蹤系統上的各種狀態，一個經典的範例就是，維運團隊透過 Excel 試算表來管理每個網路介面目前到底開啟了哪些連接埠。今天如果有人想要對連接埠的使用進行修改，必須要對該 Excel 試算表發出一個修改請求，一旦該請求通過後，就會有專屬的人員將這份修改給套用到正式機器上。


## Deployment Pipeline Created By GUI
隨者 DevOps 的概念近年成為軟體開發的標準配備，愈來愈多的團隊會透過類似 Jenkins 這種工具來打造建置與部署的流水線。 然而打造這些流水線的做法很多時候都是透過 GUI 直覺的操作，之後的更新與維護也都是基於 GUI 來完成

上述這些流程其實會導致一些維運上的問題，而這些問題就是 GitOps 想要解決的，因此接下來我們來看一下這些問題

## Uncertain State
上面這些不良的操作流程其實都會導致我們難以瞭解當前系統的狀態，舉例來說。今天有一個工程師想要對測試環境的 database 的 table schema 進行一些手動微調來處理 bug，那未來其他工程師要怎麼知道這個操作，以及其他環境要如何一起套用這個修正。
或是有一個團隊想要把其他人的流水線設計也設計一份到自己環境中，如果都依賴 GUI 來管理的話，那該怎麼達成？

## Gap Between Desired And Actual State
相同的，如果今天有任何人員的主動修改但是卻沒有任何文件紀錄，那就會產生一個系統狀態認知的差距，維運團隊期望的狀態以及正式系統運行的狀態這之間的差異會因為愈來愈多的手動介入而愈來愈大。

如果你很幸運的剛好有一些文件幫你記住當前運行的狀態，譬如上述的 Excel 試算表，那你可以幸運的解決這些問題。但是更多的情況下，很多系統上的狀態，譬如軟體版本，軟體用的設定檔案可能都沒有很好的被記錄跟追蹤。

當系統一旦發生這些情況時，往往都依賴一些經驗老道，系統熟悉的工程師幫忙整理跟善後，盡可能的解決這些問題，讓系統變成可接受的狀態。


## Control Challenges
上述這些流程還會造成一個很嚴重的問題就是，對於系統上的任何修改，非常困難的去回答
1. 為什麼會有這些修改
2. 系統發生什麼事情
3. 誰提出這個請求
上述這個需求對於很多公司來說都是必備的，但是一旦今天沒有辦法滿足這個條件，那工程團隊就要花更多的時間去解決這些未知的答案。

大型企業通常都會使用一些如 `ServiceNow` 或是 `Remedy` 之類的工具來管理整個工作流程，透過這些工具去進行一些稽核的動作來追蹤是誰執行了這些變化。
但是這些工具跟真實運行的系統其實還是有脫鉤，很多系統上的變化都沒有辦法即時的被該系統追蹤，所以就現實面來說，還是會有一些問題沒有辦法回答上述的三個問題。


## How GitOps Helps
談了這些問題後，我們來看一下 GitOps 期望怎麼解決

### Git to Provide
GitOps 希望透過 Git 這套版本系統來提供下列特性
1. 提供一個 single source of truth 的特令來維護系統的期望狀態
2. 有提供歷史紀錄可以追蹤每次的修改，包含修改內容，誰造成修改
3. 透過 git 原本的整合系統，可以提供管理機制來確保能不能修改
### Declarative data formats (such as YAML or HCL) to:
1. 所有的檔案格式都希望基於宣告式內容，用這些格式來描述系統的期望狀態
2. 透過檔案的方式來描述狀態而不是透過程式化來產生

### Control loop tools to
透過一個循環的控制邏輯將系統與 Git 連動
1. 確保系統上的運行狀態都能夠與 Git 內描述的期望狀態一致
2. 如果系統上有任何期望的修改，這些修改可以被寫回到 Git 中去保存


上述這些提到的問題，以及解決問題的方法其實早就存在，並不是一個全新的想法。
然而 GitOps 做的事情就是將這些方法集中起來，提供一個系統化的解決方法讓工程團隊能夠有一個共識來解決維運上的各種挑戰

到這邊為止，我們已經瞭解了關於 GitOps 的基本概念以及其期望解決的問題。
下一篇我們會開始探討 GitOps 理論上的定義以及實作上的細節。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/JPwq4znr1

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=7EP1KY3

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



