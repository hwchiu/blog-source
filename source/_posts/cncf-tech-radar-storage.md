---
title: CNCF Storage 使用者調查報告
keywords: 'Kubernetes,CNCF,Storage'
tags:
  - Devops
  - Storage
  - CNCF
  - Kubernetes
description: >-
  本篇文章節錄自 CNCF End User Technology Radar 關於 Storage 的報告，擷取相關重點並加上個人心得來跟大家分享現在
  CNCF 社群是怎麼選擇自己適合的 Storage 解決方案
abbrlink: 41161
date: 2021-03-13 21:53:27
---

CNCF Storage 使用者調查報告

# 前言

今天要分享的 CNCF Radar 是 2020/11 所公布的報告，該報告所瞄準的範圍是 Database Storage。

就如同前篇文章所述 CNCF Continuous Delivery 使用者調查報告， CNCF 雷達主要是針對 CNCF 會員的使用經驗進行調查，根據這些經驗回饋來統計當前 CNCF 會員對於各項解決方案的推薦程度。

本篇文章翻譯自 [Database Storage, November 2020
](https://radar.cncf.io/2020-11-database-storage)，並且加上個人心得

詳細的訪談影片可以參閱 [CNCF End User Technology Radar: Database Storage, November 2020
](https://www.youtube.com/watch?time_continue=151&v=Ypq9P-9WBQI&feature=emb_logo)


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

這次的報告總共有來自 26 個 CNCF 會員參與，全部票數有 273 票，參與的廠商規模有大小，領域也不同，下圖節錄自官方報告

![](https://i.imgur.com/7gdQkCB.png)

從人數規模來看，基本上每個公司都是百人規模以上，甚至一半以上都是千人等級，還有六家公司是萬人等級。

# 報告
下圖節錄自官方的結論報告
![](https://i.imgur.com/ExWvRdH.png)

該報告就是根據上面的標準，讓參與的 CNCF 會員來回報對這些專案的推薦程度

> 這邊要注意，這邊的結果是粗略的統計結果，沒有太多明確的數學定義到底什麼樣的等級可以歸類為 ADOPT，所以觀看時就當做一個參考看看即可

![](https://i.imgur.com/ZVY4Nd9.png)

上述的統計報表中，可以觀察到一些資訊
1. Redis, Elasticsearch, PostgreSQL 這三個專案的推薦程度都是一致性的高，幾乎是有使用過的團隊都會推薦
2. MySQL, MongoDB, Cassandra 這三個專案獲得較多的 `Assess` 票數，也就是有不少團隊使用後並沒有強烈推崇繼續使用，反而保持觀望的角度。
3. Kafka, Memcached 等知名專案也都有很高的關注度及支持度
4. AWS 底下的 DynamoDB 以及 Aurora 都有上榜，算是雲端廠商內提供這塊服務做得最好的


# 結論
1. Companies are cautious with their data and slow to adopt newer technologies.
> 團隊傾向謹慎且小心地去導入全新的儲存技術來處理資料，調查報告中的新專案如 CockroachDB, TiDB, Vitess 都沒有於調查報告中獲得廣大的迴響。

> 有很多理由使得團隊會謹慎小心的去採用新儲存技術，其中最主要的原因就是太難去管理。當團隊需要轉移大量資料，從數 TB 到 PB 這種層級時，這中間花費的成本非常巨大，因此轉移帶來的好處要是不能夠蓋掉轉移的成本的話，很難說服團隊去進行儲存專案的轉移

> 其他導致謹慎選擇的可能原因是很難找到一個新專案技術的專家

> 令人感到興趣的是知名的 etcd 竟然沒有出現在報告中。大部分 etcd 的使用都是基於 Kubernetes 的需求。看起來很少團隊會單獨使用 etcd 來管理其資料。

2. Choosing a managed database service depends heavily on use cases.
> 報告中檢視使用雲端管理的儲存技術得票數並不高，這令人感到驚訝。一種解讀方式是雲端管理的資料技術非常仰賴使用情境，譬如應用程式最終部署的位置，資料的使用量以及團隊是否本來就使用該雲端產品。

> 如果應用程式產生的資料非常大量，使用這些雲端管理的儲存技術可能會帶來非常巨大的花費

> 是否使用雲端管理的儲存設備一個很大的影響原因是該團隊是否已經使用該雲端服務。假如一個團隊所有服務都是地端自行架設，不太可能單獨將資料放到雲端去保存。

> 團隊如果處理的是非常敏感機密的資料的話，更有可能將資料給部署到自己的機器上而非使用雲端管理的儲存設備。

> 訪談中談到 AWS RDS 這個服務，其獲得的票數也非常的少所以最終被移除出報告之中。
3. Keep an open mind!
> Storage 領域依然處於一個開發演化的階段，可以發現到很多專案都是非常久遠且知名的大專案。這些專案藉由其穩定且運作來好的特性帶來了很高量的 ADOPT 票數

> 愈來愈多針對 Cloud Native 環境下所開發的儲存專案，而這些專案都有專屬的使用情境使得很難落入使用者雷達這種比較全面性的調查報告中。


# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

詳細可以參閱
矽谷年線上學院https://course.hwchiu.com
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
