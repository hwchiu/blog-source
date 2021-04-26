---
title: CNCF Secrets 使用者調查報告
keywords: 'Kubernetes,CNCF, Secrets'
tags:
  - Devops
  - Security
  - CNCF
  - Kubernetes
description: >-
  本篇文章節錄自 CNCF End User Technology Radar 關於 Secret的報告，擷取相關重點並加上個人心得來跟大家分享現在 CNCF
  社群是怎麼選擇自己適合的 Secret 管理工具
abbrlink: 46591
date: 2021-03-30 00:11:30
---

CNCF Storage 使用者調查報告

# 前言

今天要分享的 CNCF Radar 是 2021/02 所公布的報告，該報告所瞄準的範圍是 Secret Management 。

就如同前篇文章所述 CNCF Continuous Delivery 使用者調查報告， CNCF 雷達主要是針對 CNCF 會員的使用經驗進行調查，根據這些經驗回饋來統計當前 CNCF 會員對於各項解決方案的推薦程度。

本篇文章翻譯自 [Secret Management, February 2021](https://radar.cncf.io/2021-02-secrets-management)，並且加上個人心得

詳細的訪談影片可以參閱 [CNCF end user technology radar, February 2021 - Secrets Management](https://www.youtube.com/watch?v=sUC04b_gh-Y)

Secret Management 對於要導入自動部署的團隊來說是個不可避免的挑戰，就算使用 GitOps 流程也有一樣的問題。
原文所提及的 Secret Management 不單單只是應用程式的機密資訊，也包含如何管理系統中使用的大量憑證。



# Radar
Technology Radar 旨在成為特定領域的一個意見參考指南，因此 CNCF End User Techonlogy Radar 就是一個針對科技領域受眾所建立的一個專案參考指南，這些專案領域都聚焦於 Clou Native 上，透過這個報告可以知道 CNCF End User Community 內這些公司他們實際上都使用哪些解決方案，對於這些方案保持什麼樣的看法



# Level
為了簡單量化這些調查報告，所有的調查都會要求使用者對於是否推薦這個專案給予下列答案之一

- Adopt
這個答案代表該使用者(通常是廠商)是明確的推薦這個技術，使用者已經使用這個專案一段時間，而且也被團隊內證實的確是穩定且有幫助的
- Trail
這個答案代表使用者有成功的使用過這些技術且推薦大家要多關注這些技術的發展
- Assess
這個答案代表使用者有嘗試使用過且認為他們是有未來的，推薦大家當你專案內有特別需求的時候可以去看看這些專案
基本上我的認知就是信心程度，由上到下遞減。

除了上述三個答案之外，還有一個選項就是 HOLD，顧名思義就是可以停一下，不要太執著這個專案甚至不要使用。

關於這個專案的一些運作，譬如題目跟專案的選擇，甚至一些概念的介紹都可以參閱[官方網站](https://radar.cncf.io/how-it-works)

# 資料來源

這次的報告總共有來自 29 個 CNCF 會員參與，全部票數有 79 票，參與的廠商規模有大小，領域也不同，下圖節錄自官方報告

![](https://i.imgur.com/7gdQkCB.png)

從人數規模來看，基本上每個公司都是百人規模以上，甚至一半以上都是千人等級，還有六家公司是萬人等級。

# 報告
下圖節錄自官方的結論報告
![](https://i.imgur.com/2VNIv09.png)


該報告就是根據上面的標準，讓參與的 CNCF 會員來回報對這些專案的推薦程度

> 這邊要注意，這邊的結果是粗略的統計結果，沒有太多明確的數學定義到底什麼樣的等級可以歸類為 ADOPT，所以觀看時就當做一個參考看看即可

![](https://i.imgur.com/8AWYFwV.png)


上述的統計報表中，可以觀察到一些資訊
1. Hashicorp Vault 可以說是自架解決方案的霸主，第二名 Sealed Secrets 可能算看不到車尾燈
2. 管理憑證方面幾乎是 Cert-manager 的天下
3. AWS 提供的兩個工具, AWS Secrets Manager 以及 AWS KMS 也都上榜，是公有雲廠商中最受青睞的。
4. GCP Secrets Management 可以看到很多團隊使用後就保持觀望

# 結論
1. Vault has the broadest adoption across many companies and industries
> HashiCorp Vault 被廣泛地使用於各公司以及各領域中

> Vault 過往總是被認為是一個複雜且難以駕馭維運的工具，但是對於很多小組織與團隊來說，與其外包 Secret Managment 服務或是自己設計相關服務，使用 Vault 反而是一個折衷的方式，其解決了自行設計解決方案的困難處同時又可以自行維護。

> Vault 可能會這麼受歡迎的原因是因為其解決 secret management 的方式不會很綁死雲端環境，使用各種雲端服務的團隊都可以導入 Vault 到其環境中。


2. After Vault, groups tend to use the native solutions provided by their public cloud provider.

> 這個結果是顯而易見的，對於很多團隊來說，如果已經大量採用某雲端技術，通常也會考慮直接使用整合好的 secret management 服務

> Radar 的資料報告中被提及的雲端解決方案有 AWS Secrets Manager, AWS Key Management Service, AWS Certificate Manager, Azure Key Vault, GCP Secrets Management.
不過最後 Azure Key Vault 以及 AWS Certificate Manager 都沒有上榜

> 根據 Radar 的報告顯示，基礎建設的規模程度以及使用者數量都會大大的去影響團隊如何選擇這些 secret management 的解決方案。特別是當要考慮使用多雲環境時，如果將使用特定雲端廠商的服務很容易陷入 vendor lock-in 的狀態，導致未來要搬遷轉移都很困難。


3. Certificate manager has become a popular choice in the Kubernetes ecosystem.
> Certificate manager 於 certificate 這方面的處理獲得極高的使用率
> 其本身的使用情境不是廣泛使用，專注於 certificate 方面的處理，同時與 Kubernetres 有極好的整合性

4. Other solutions in the space are fragmented across various levels of maturity and complexity.
> 除了上述幾個大方向外， CNCF 團隊也觀察到 Radar 中有非常多小量零星的專案，這些專案都被設計成更明確的使用方向，譬如開發框架，Git 專案中加密或是已經跟 Kubernetes 生態系中有大量重疊的使用情境。 這些工具因為沒有收到足夠數量的回應所以都沒有上榜



# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

詳細可以參閱
矽谷牛線上學院
https://course.hwchiu.com

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
