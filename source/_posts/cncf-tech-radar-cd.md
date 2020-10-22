---
title: CNCF Continuous Delivery 使用者調查報告
keywords: 'Kubernetes,CNCF, CD, Continuous Delivery'
tags:
  - Devops
  - GitOps
  - CNCF
  - Kubernetes
description: >-
  本篇文章節錄自 CNCF End User Technology Radar 關於 Continuous Delivery
  的報告，擷取相關重點並加上個人心得來跟大家分享現在 CNCF 社群是怎麼選擇自己適合的 CD 工具
abbrlink: 44745
date: 2020-09-12 01:25:20
---


# 前言
今天要分享的 CNCF CD 使用者調查報告是由  CNCF End User Technology Radar 所發佈的，該專案會定期針對不同領域發布一些不同領域的研究，調查 CNCF 的會員針對不同的軟體有什麼樣的想法，因此這篇文章就帶大家來看一下這篇關於 Continuous Delivery 的報告，來看看這些 CNCF 使用者們都使用哪些工具，對於這些工具又有什麼評價

本文主要翻譯自 [Continuous Delivery, June 2020
](https://radar.cncf.io/2020-06-continuous-delivery)，並且加上個人心得

詳細的報告影片可以參閱 [Webinar: What end users really recommend for Continuous Delivery
](https://www.youtube.com/watch?time_continue=832&v=0792Q3l6tK8&feature=emb_logo)

# Radar
Technology Radar 旨在成為特定領域的一個意見參考指南，因此 CNCF End User Techonlogy Radar 就是一個針對科技領域受眾所建立的一個專案參考指南，這些專案領域都聚焦於 Clou Native 上，透過這個報告可以知道 CNCF End User Community 內這些公司他們實際上都使用哪些解決方案，對於這些方案保持什麼樣的看法

## Level
為了簡單量化這些調查報告，所有的調查都會要求使用者對於是否推薦這個專案給予下列答案之一
1. Adopt
這個答案代表該使用者(通常是廠商)是明確的推薦這個技術，使用者已經使用這個專案一段時間，而且也被團隊內證實的確是穩定且有幫助的
2. Trail
這個答案代表使用者有成功的使用過這些技術且推薦大家要多關注這些技術的發展
3. Assess
這個答案代表使用者有嘗試使用過且認為他們是有未來的，推薦大家當你專案內有特別需求的時候可以去看看這些專案

基本上我的認知就是信心程度，由上到下遞減。

除了上述三個答案之外，還有一個選項就是 `HOLD`，顧名思義就是可以停一下，不要太執著這個專案甚至不要使用。

關於這個專案的一些運作，譬如題目跟專案的選擇，甚至一些概念的介紹都可以參閱[官方網站](https://radar.cncf.io/how-it-works)


# Continuous Delivery

## 資料來源
上述影片中說有 33 個廠商回報，但是根據網頁的結果大概只有 28 個，所以實際上到底會有多少我不確定，不過大概就是 30 上下左右。

這些公司散落於不同產業，同時公司內的員工數量也不一定，這些數據都可以從官方網頁中看到，如下圖
![](https://i.imgur.com/WAw8IX5.png)

![](https://i.imgur.com/CeRR65O.png)



## 報告
下列就是報告的 Radar 圖

![](https://i.imgur.com/5cY59sh.png)
這邊就是根據最上面的三個標準，讓參與的 CNCF 使用者廠商來回報對這些專案的想法。


> 這邊要注意： 這邊的結果是粗略的統計結果，沒有太多明確的數學定義多少廠商回報 ADOPT 就可以歸類為 ADOPT，就當過一個普遍的趨勢去看待就好。

從上圖可以看到，目前被歸類為 ADOPT 的有兩個專案，分別是 Helm 以及 Flux
Helm 大家應該都很熟悉， Kubernetes 應用程式的打包及部署解決方案，而 Flux 則是一個基於 GitOps 所實作的 CD 解決方案。

以下是根據影片內投影片的結果來統計其每個 Level 的數量

|--| Adopt | Trial | Assess | Hold |
|--| -------- | -------- | -------- | ---- |
|Flux | 7     | 1     | 3     | 0 |
|Helm | 12     | 4     | 3     | 0 |



接下來看到 TRIAL 這列表，包含了
1. Circle CI
2. Kustomize
3. GitLab

|--| Adopt | Trial | Assess | Hold |
|--| -------- | -------- | -------- | ------|
|Circle CI | 5     | 0     | 1     | 3 |
|Kustomize | 5     | 3     | 3     | 1 |
|GitLab | 4     | 2     | 3     | 0 |

最後看到 Assess 列表，會被放到 Assess 的專案通常都是大家沒有一個很明確的共識。
這邊的專案很多，包含

1. ArgoCD
2. Jenkins X
3. Spinnaker
4. TeamCity
5. GitHub Actions
6. Tekton CD
7. Travis CI
8. Jenkins
9. jsonnet

|--| Adopt | Trial | Assess | Hold |
|--| -------- | -------- | -------- | ------|
|ArgoCD | 2     | 3     | 10     | 0 |
|Jenkins X | 0     | 1     | 3     | 2 |
|Spinnaker | 0     | 3     | 6     | 4 |
|TeamCity | 1     | 0     | 0     | 1 |
|GitHub Actions | 2     | 7     | 7     | 0 |
|Tekton CD| 0     | 0     | 2     | 0 |
|Travis CI| 1     | 1     | 1     | 2 |
|Jenkins  | 6     | 0     | 2     | 17 |
|jsonnet| 1     | 1     | 2     | 2 |

這份清單的確可以看到大部分的專案都沒有明顯的共識，有些專案可能使用者數量也少，導致回應的統計數量也少。
比較有趣的點有
1. 喜愛 Jenkins 的是裡面最多的，但是認為不要碰 Jenkins 的也是最多的，更甚一個量級，比 Adopt 的 Helm 支持者還要...
2. GitHub Action 跟 ArgoCD 看起來算是相對獲得好評，本身沒有任何 HOLD 的紀錄
3. Jenkins X / Teckon CD 這兩套看起來使用者還不算太多
4. jsonnet 相對於 Kustomize 以及 Helm 還是小眾

## 結論
1. Publicly avaliable solutions are combined with in-house tools
根據調查結果，大部分的使用者都嘗試過至少十套以上的解決方案，最後收斂到2-4套左右去使用。有些大公司會建置自己的 CD 工具並且開源部分，譬如 [release-manager](https://github.com/lunarway/release-manager), [kube-applier](https://github.com/box/kube-applier) 以及 [stackset-controller](https://github.com/zalando-incubator/stackset-controller)。
另外一個有趣的是沒有一個公有雲廠商的 CD 解決方案被使用者給建議，這部分是個有趣的結論。

2. Helm is more than packaging applications.
雖然 Helm 不會被認為是一個 CD 工具(其本質更像是一個 Kubernetes 應用程式管理者)，但是 Helm 卻是一廣泛被應用於各種 CD 解決方案中的管理工具。

3. Jenkins is still broadly deployed, while cloud native-first options emerge.
Jenkins 以及其相關生態系的相關工具 (Jenkins X, Jenkins Blue Ocean) 還是被廣泛地使用。然後滿多廠商都表示 Jenkins 主要是用在存在已久的系統，如果是全新的應用環境，都會嘗試使用其他的解決方案，譬如更加支援 GitOps 的 Flux 等

# 個人心得
這個專案我認為算滿有趣的，透過 CNCF 的使用者社群來調查大家使用過哪些工具，對於這些工具有什麼想法，可以整理出對於滿山滿谷專案下，一個普遍的共識（或是沒有共識）。

事實上這個專案調查的時候也沒有辦法列出世界上所有的 CD 解決方案，所以可能會有些專案沒有被列入考慮中，但是我認為這類型的報導跟調查都還是有其價值所在，雖然受訪群體不多，就 30 多個廠商為主，但是畢竟是以廠商為單位，不是以人為單位，本來就很難動不動就湊到幾千幾萬個人來回報資料。

就結果的分析來看， Helm 這個工具還是有其地位所在，再來就是 Kustomize，兩者我都喜歡，畢竟各有各適合的場景。
對於系統來說, Jenkins 真的是一個又愛又恨的選項，彈性高，維護麻煩，除此之外，可以看到也有滿多開源專案跟 SaaS 平台上榜，其中 Flux/ArgoCD 的排名有點出乎我意料就是了，比我想像多的公司在使用 GitOps 類似的部署流程


# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/mLwKb3zrx

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4XNG

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


# Reference
1. https://radar.cncf.io/2020-06-continuous-delivery
2. https://www.youtube.com/watch?time_continue=832&v=0792Q3l6tK8&feature=emb_logo
3. https://radar.cncf.io/how-it-works
