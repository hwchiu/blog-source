---
title: '淺談 Kubernetes 設計原理'
keywords: 'Kubernetes, design, CNI, CSI, CRI'
abbrlink: 51275
date: 2019-09-16 06:17:10
tags:
  - Kubernetes
  - ITHOME
  - CRI
  - CSI
  - CNI
description: 為了跟大家分享 kubernetes 內部的幾個重要設計，如 CRI, CSI 以及 CNI, 本篇文章先簡單介紹了一下 kubernetes 內部相關的設計理念，透過理解這些理念更可以理解為什麼會有各式各樣的介面被設計出來。
---

# 前言


`Kubernetes` 作為近年來討論度最高的容器管理平台，從自行架設，使用公有雲相關服務甚至到尋求第三方廠商解決方案都已經是日常可見的作法。

使用場景來看，各式各樣的場景都在思考與評估是否有機會將 `kubernetes` 納入其應用範圍，從架設服務器提供服務，配合 GPU 進行大量運算使用甚至將 `kubernetes` 與 5G網路產業結合。 各式各樣的需求不停的發出，而 `kubernetes` 是否能夠滿足這些所有的需求則是一個需要好好思考的問題。

為了評估到底 `kubernetes` 能不能適用於各種使用情境，我們必須先知道什麼是
`kubernetes` 的極限，我認為透過學習其實作原理與設計理念能夠提供一個基本的能力去評估到底 `kubernetes` 能不能滿足所需。

接下來的系列文內，我會針對 `kubernetes` 內幾個最大的特點也是所有使用情境最需要考慮的幾個方向來探討，如下

- 運算單元
- 網路架設與連線
- 儲存空間
- 其他特殊裝置

藉由學習這些不同面向功能的實作原理與設計開發理念，我們都能夠有更好的立足點去評估到底 `kubernetes` 是否能夠滿足目前所需，甚至說若需要進行第三方開發改善時，該怎麼下手。


# 架構

`Kubernetes` 作為一個開放原始碼的專案，其所有原始碼都可以在 [Github](https://github.com/kubernetes/kubernetes) 看到，同時也可以在看來自世界各地的使用者與開發者如何合作一起開發這個巨大的專案。

對於一個成熟的開源軟體來說，通常都會有所謂的 Release 版本週期，並不是每個開發的新功能都可以很準時的被安排在下一個版本釋出，同時也意味有時候希望某些新功能可以順利的使用，都要等到下個 Release 週期釋出，否則就要自己透過版本控制的方法去 `build/compile` 來使用最新的功能。

基於以上的軟體開發流程情況，可以試想一下一個情境。
今天某開發者開發了一個有趣的功能，吸引很多使用者都想要趕快嚐鮮使用，然後基於上述的理由，該功能要先經過整體的程式碼評估與測試，最後合併。接者還要等上一段時間產生一個正式公開的建置版本，這一切跑完可能都是數天，數週，甚至數個月後的事情。

而對於 `Kubernetes` 來說，所謂的 `容器管理平台` 其涉略的領域實在太多，對於 `kubernetes` 的眾多開發者來說，要能夠完全掌握這些不同領域的技術與概念其實也是很困難的事情。

舉例來說
今天有一個熱心的開發者想要讓 `Kubernetes` 支援 `GPU` 的運算，於是提交了相關的程式碼改動， 如果 `kubernetes` 的維護者對於 `GPU` 的運作原理不夠掌握是否有辦法幫他進行程式碼的審查?

同時加上 `kubernetes` `release` 週期的規則，會使得這些由來自世界各地貢獻者的結晶沒有辦法很即時的被一般使用者與測試。

整個運作流程如下圖
![](https://imgur.com/VFxfxpr.png)


為了使得整體的開發流程更加順暢，如果能夠針對架構比較需要彈性的功能進行架構改造，將介面與實作給獨立出來各自運作。此架構中， `kubernetes` 只要設計介面，並且專注於本身與介面的溝通與整合，第三方的開發者則是專注於開發應用，只要該應用符合介面標準即可。

這種狀況下，第三方的開發者可以自行決定其軟體/產品的 `release` 週期，不需要與 `kubernetes` 本身掛鉤。不但能夠讓整個平台的擴充功能開發更佳流暢，同時使用者也可以更方便地去嘗試各種不同的底層實現。


改良後的架構可參考下圖，`kubernetes` 與其他各自的解決方案可以有自己的開發週期
與流程，彼此之間透過事先定義好的介面進行溝通，如此一來就可以提升整體開發的
流暢度。
![](https://imgur.com/FcbTSDc.png)

接下來的29天，會帶領讀者一起探討這些介面的設計以及各種不同應用的實作。
包含了
- 基於運算單元的 `CRI (Container Runtime Interface)`,
- 提供平台容器網路連接能力的 `CNI(Container Network Interface)`，
- 提供儲存能力供容器使用的 `CSI (Container Storage Interface)`
- 以及可以掛載各式各樣系統裝置的 `Device Plugin`.

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
