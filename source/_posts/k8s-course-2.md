---
title: 'Kubernetes 線上課程'
tags:
  - Kubernetes
  - Course
keywords: 'Kubernetes,Network,Linux,Ubuntu'
abbrlink: 1535
date: 2020-04-23 20:51:23
top: 100
description: Kubernetes 線上課程分享, 從底層到應用來解決你對 Kubernetres 的各種疑惑

---

Kubernetes 與 Docker 非常不一樣，其複雜程度不是一個量級可以比擬的，過往單節點的部屬使用 docker 或是 docker-compose 就可以輕鬆處理，然而到了
多節點狀況下，支援的功能更多，相對的也有更多的問題需要預防與處理，總總原因使得 Kuberntees 學習曲線非常大，光要建置第一個測試環境就有非常多的
先行概念要理解。
過往學習 Docker 時，可能只要用一行 Docker run 就可以跑起基本範例，然而到 Kubernetes 裡面，除了要架設基本叢集，裡面還有各式各樣的資源需要學習
只會 Docker 的情況下會發覺很難上手，第一步都不確定該怎麼踏出去。

然而當你學會基本的 Kubernetes 使用後，接下來的問題就是要如何將團隊內的服務都導入到 Kubernetes，甚至整合 CI/CD 流程到 Kubernetes 內。這部份的過程更加繁瑣，從本地開發，如何打包 Kuberentes 應用程式到如何設計與建置 CI/CD 流水線，最後透過不同的部屬方式來更新 Kubernetes 內的應用程式，每個都有其很深的坑
需要去踩，去學習。

當一切都準備完畢就緒後，接下來就是要如何好好的維護 Kubernetes，遇到問題時該怎麼除錯，如何用最快最有效的方式找到問題點，這些對於生產環境來說都是非常重要的課題。

我用我這幾年來對 Kubernetes 的實戰經驗，搭配我對 Kubernetes 原始碼的理解，整理出一系列學習 Kubernetes 的課程，課程中更加強調的是教你釣魚，而不是給你魚吃，所以每個章節都會從原理，從概念，從設計出發，透過瞭解這些基本原則，你去學習 Kubernetes 的時候會更加體會一切設計的根本，同時未來碰到各式各樣的新專案的時候，也能夠游刃有餘的去拆解其概念併整合到 Kubernetes 內。

目前這系列的課程總共有三堂課程，分別是
- Kubernetes 基礎課程，該課程會從基本談起，為什麼需要 Kubernetes，其與 Docker 的差異是什麼，Kubernetes 能夠解決什麼問題，並且從安裝，運算，網路，儲存等四大面向來談起，讓你學起來更加得心應手，可以知道其背後原理，對於以後觀看任何文章都能夠更有共鳴

- Kubernetes CI/CD 整合，本課程探討如何將 CI/CD 的概念與 Kubernetes 整合，會從一個完整的 CI/CD 流水線去看，從最初的本地開發，應用程式打包，CI 可以怎麼做，CD 可以怎麼做，並且探討不同的 CI/CD 方式。透過理解這些方式後，接下來使用各種不同的工具都不會覺得太困難，因為我們掌握的是分析問題的能力
並且從問題中學習到自己需要什麼，再由這個需求找到正確工具來整合。

- Kuberentes 實戰除錯篇，本課程基於 Kubernetes 龐大架構內，實戰上才遇到的迷思與錯誤，每一個章節都有完整的 Demo 環境，讓你知道每個元件出錯時可能會造成什麼樣的後果，或是這些元件應該要怎麼用，使用上有什麼限制以及要注意的地方。透過本課程你會對 Kubernetes 更加裡解，同時使用上會更有信心與能力去使用，縱使遇到問題時也會有能力知道該從哪邊下手去判斷錯誤。

以下是課程相關的購買連結，歡迎有興趣學習的朋友支持支持，或是幫忙分享:)

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE

疑難雜症除錯篇

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
