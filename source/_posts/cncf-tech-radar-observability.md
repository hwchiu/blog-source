---
title: CNCF Observability 使用者調查報告
keywords: 'Kubernetes,CNCF, Observability'
tags:
  - Devops
  - Observability
  - CNCF
  - Kubernetes
description: >-
  本篇文章節錄自 CNCF End User Technology Radar 關於 Observability
  的報告，擷取相關重點並加上個人心得來跟大家分享現在 CNCF 社群是怎麼選擇自己適合的 CD 工具
abbrlink: 9340
date: 2020-10-13 20:25:39
---


# 前言
今天要分享的 CNCF Radar 是 2020/09 所公布的報告，該報告所瞄準的範圍是 Observability(可觀測性)。
就如同前篇文章所述 [CNCF Continuous Delivery 使用者調查報告](https://www.hwchiu.com/cncf-tech-radar-cd.html)， CNCF 雷達主要是針對 CNCF 會員的使用經驗進行調查，根據這些經驗回饋來統計當前 CNCF 會員對於各項解決方案的推薦程度。

本篇文章翻譯自 [Observability, September 2020](https://radar.cncf.io/2020-09-observability)，並且加上個人心得

詳細的訪談影片可以參閱 [Webniar: CNCF End User Technology Radar, Sep 2020: Observability](https://www.youtube.com/watch?time_continue=2039&v=I44EepyZGNo&feature=emb_logo)

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

# Observability

## 資料來源
這次的報告總共有來自 32 個 CNCF 會員參與，全部票數有 283 票，參與的廠商規模有大小，領域也不同，下圖節錄自官方報告

![](https://i.imgur.com/j0nZGIh.png)

從人數規模來看，基本上每個公司都是百人規模以上，甚至一半以上都是千人等級，還有六家公司是萬人等級。
這數字我個人認為台灣很難找到如此規模的公司再探討 CNCF 可觀測性的應用，此外這些公司裡面，大部分都來自於軟體公司

## 報告

下圖節錄自官方的結論報告
![](https://i.imgur.com/mge9B8x.png)

該報告就是根據上面的標準，讓參與的 CNCF 會員來回報對這些專案的推薦程度

> 這邊要注意，這邊的結果是粗略的統計結果，沒有太多明確的數學定義到底什麼樣的等級可以歸類為 ADOPT，所以觀看時就當做一個參考看看即可

# ADOPT
該圖片中，歸類為 ADOPT 也就是非常推薦使用的解決方案有五個，分別是

1. Elastic
> Elastic 這邊沒有說明是開源專案還是商業解決方案，畢竟 Elastic 實際上還包含了很多專案一起，常用的 ELK 可以算是其中之一。
1. Datadog
> Datadog 本身是一個商業解決方案，提供客戶一種視覺化的服務來監控與觀測系統上的各式各樣資料，這公司自疫情以來，股價已經翻了三倍，財報屢屢創新高
1. Prometheus
1. OpenMetrics
1. Grafana
> 這三個幾乎可以一起談，Prometheus 以及 Grafana 這兩套軟體大家使用上都會一起使用，很少看到單獨使用的。 透過  Prometheus 的介面，可以串皆各式各樣的 Metrics 並且透過 Grafana 來將這些資訊用自己喜歡的方式呈現
> OpenMetrics 本身也是 CNCF 的專案之一，其目的主要是探討 Metrics 的格式，希望透過制定標準來讓各解決方案輕鬆整合，詳細的介紹可以參閱這個 [CNCF to host OpenMetrics in the Sandbox
](https://www.cncf.io/blog/2018/08/10/cncf-to-host-openmetrics-in-the-sandbox/)

# TRIAL
這邊總共有六個工具，代表的是有使用，並且強烈推薦觀望其發展，認為其有使用的潛力，這邊包含了
1. Splunk
> 商業的日誌收集解決方案，算是非常老牌的服務，印象中價格會根據日誌容量來計費，所以如果今天服務開啟了大量的 debug模式的話，可能開銷會突然增加不少
1. Sentry
> 一套針對應用程式的觀測與除錯解決方案，使用起來非常方便且好用，特別是當出現問題的時候能夠提供更多友善的資訊幫忙除錯，我個人是滿喜歡的。
1. Cloudwatch
> AWS 內建的觀測平台，這部份就沒有什麼特別好說，我想應該不會有人沒有使用 AWS 卻跑來使用這套系統
1. Lightstep
> 查了一下也是一套商業解決方案，本身並沒有使用過的經驗，所以也不好說
1. Statsd
> 個人沒有聽過也沒有用過
1. Jaeger
> 作為 Uber 所開源的 Opentracing 解決方案，我之前有一個影片詳細介紹 Opentracing 與 Jaeger，有興趣的可以觀看 [SDN x Cloud Native Meetup - Webinar 邱牛上菜 #3 OpenTracing
](https://www.youtube.com/watch?v=t8OCKZYcVLg) 

## ASSESS
這個類別中只有三個選項，分別是 OpenTelementry, Thanos, Kiali ，代表這些專案是有使用過，且認為不錯，但是只有真的需要的時候才特別需要去研究。


1. OpenTelementry
> 這部份我個人認為可能是個趨勢，該組織已經將 OpenTracing 給整合進去，希望能夠提供一個更為通用的函式庫以及介面來使用，就我所知 Jaeger 的部份程式碼都已經被整合進去。 詳細的也可以觀看  [SDN x Cloud Native Meetup - Webinar 邱牛上菜 #3 OpenTracing
](https://www.youtube.com/watch?v=t8OCKZYcVLg) 這個影片，最後面有跟大家分享 OpenTracing, Jaeger, OpenTelementry 三者的差異

1. Thanos
> 作為 Prometheus HA 的解決方案，就我所知用過的人都覺得還不錯，除了這個解決方案之外，不確定還有什麼好方式可以幫 Prometheus 搭建起 HA 的環境
1. Kiali
> 作為一個 Service Mesh Istio的管理工具，我本身是沒有使用經驗，所以也無從判斷

最後用長條統計圖來再次觀看一下三個類別的資訊
![](https://i.imgur.com/mBVVlFC.png)


# 結論

文章與影片中，針對這些報告給了三個結論，這邊簡單節錄部分內容，有興趣的可以觀看 [原文](https://radar.cncf.io/2020-09-observability)

1. The most commonly adopetd tools are open source.
> 整份報告裡面，三個收集到最多 `Adopt` 投票回饋的工具 (Prometheus, Grafana, Elastic) 以及五個收集到最多投票回饋的工具 (Prometheus, Grafana, Elastic, Jaeger, OpenTelemetry) 全部都是開源軟體。
作者認為這個議題滿有趣，可以看到大部分的公司都決定自己去維護這些開源軟體，從佈署，維護甚至支持更大規模的挑戰全部都自己處理。這些公司想必也是有針對商業解決方案去進行一些探討，並且從中比較各自的優缺點，最後才選擇自行架設使用


1. There's no consolidation in the observabilibty space.
> 大部分的公司都使用非常多的工具於可觀測性這個領域，有超過一半的公司使用五個工具以上，還有 33%左右的公司擁有十個工具以上的使用經驗
> 可觀測性這個議題其實非常廣泛，每個人使用時想要獲得的資訊都不同，同時每個工具的強項也都不同，這可能也是造成這個領域並沒有一個主宰的工具，反而是群雄割據。 此外對於大部分的使用者來說， 可觀測性並不是整體供的核心服務，因此可能也不會有太多的資源讓團隊去研究如何整合切換，這也可能就是為什麼會同時使用多套解決方案的理由之一

1. Promethesu and Grafana are frequently used together.
> 報告中表示，有超過 66% 的使用者是同時使用這兩套解決方案的。市面上有很多的教學文章或是解決方案都是將兩者整合，讓使用者可以很輕鬆的同時使用兩者

# 個人心得
我個人滿喜歡 CNCF 技術雷達的文章，可以作為一個參考來看看各領域當前主流的用法有哪些，雖然主流不代表正確，但是這也是一個信心支撐的來源，至少你有機會跟別人說，這樣用法很常見，還有 CNCF 的
文章可以背書。不過重要的是自己的團隊適合什麼工具。

就像文章中有提到，可觀測性是一個非常廣泛的議題，你要收集什麼資料，想要用這些資料回答什麼問題，再套用這些工具之前一定要先想清楚這個問題，從這個問題在去發想自己缺乏什麼工具，以及有哪些工具可以解決這些問題。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/D7RZGWrNK

單堂(CI/CD)
https://hiskio.com/courses/385?promo_code=13K49YE&p=blog1

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://www.facebook.com/technologynoteniu

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>



# Reference
1. https://radar.cncf.io/2020-09-observability

