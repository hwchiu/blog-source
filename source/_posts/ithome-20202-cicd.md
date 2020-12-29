---
title: '鐵人賽系列文章 - DevOps 與 Kubernetes 的愛恨情仇'
keywords: 'Kubernetes,DevOps,Linux,K8s'
tags:
  - ITHOME
  - DevOps
  - Kubernetes
description: ITHOME-2020 系列文章
abbrlink: 58377
date: 2020-10-21 19:19:06
---

這次的鐵人賽的題目則著重於 Kubernetes 與 DevOps 之間的關係， DevOps 這詞發展多年以來，似乎成為一個顯學
每間公司都會朗朗上口需要招聘專門負責 DevOps 的人，工作內容百百種，實在讓人難以一言就斷定到底什麼是 DevOps。

本次系列文則不會針對 DevOps 去進行細部探討，到底 DevOps 的日常生活中可能會有哪些事情要處理，相反的
本系列文主要會針對 `CI/CD` 的流程去探討，看看當 CI/CD 與 Kubernetes 整合的過程中，要怎麼處理

舉例來說，一個最大的差別就是當所有應用程式都容器化後，要如何透過 `CI/CD` 流水線將新版本部署到 Kubernetes 之中
這中間有哪些議題可以探討，針對不同議題有哪些解決方案可以使用，以及這些解決方案彼此的優缺點

下圖是一個參考流程，從開發階段到部署階段中， Kubernetes 可能會扮演哪些角色，而 `CI/CD` 流水線則會跟哪些角色有所互動

![img](https://i.imgur.com/MhJGAMt.jpg)



上次的流程中有非常多的環節可以探討，每個環節中都有不同的解決方案與取捨，這些歡節包含

1. Kubernetes 內的應用程式該如何包裝? 原生 Yaml 還是 Helm?
2. 本地開發者需要 Kubernetes 來測試 Kubernetes 嗎?
3. CI 流水線系統要選擇哪一套? 流水線工作要如何被觸發？
4. CI 流水線過程中，需要 Kubernetes 來測試應用程式嗎 ?
5. Container Registry 要使用雲端服務還是要自架，自架的話該怎麼使用以及如何與 Kubernetes 整合
6. CD 流水線系統要選擇哪一套? 流水線工作要如何被觸發?
7. CD 流水線過程中，要怎麼將應用程式更新到遠方 Kubernetes? 雲端架構或是地端架構會有什麼差異
8. CD 更新過程中，如果有機密資料，該怎麼處理



接下來的章節會針對上述環節進行介紹，每個環節都會透過下述流程來介紹

1. 概念介紹
2. 相關專案介紹
3. 使用範例



再次強調，上面的流程圖不是一個唯一的流程圖，而是一個範例，真正的運作流程都會根據不同的環境與需求而有所差異
但是選擇跟設計解決方案的思路則是不變的，透過培養思考的能力，才能夠遇到任何環境都有辦法構建出一套符合的解決流程。


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
