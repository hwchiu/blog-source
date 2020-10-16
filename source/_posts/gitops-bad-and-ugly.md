---
title: GitOps 帶來的痛點與反思
keywords: 'Kubernetes,DevOps, GitOps'
tags:
  - GitOps
  - DevOps
  - Kubernetes
description: >-
  本文翻譯自 Container-Solutions 的文章，探討 GitOps
  實際操作上可能帶來的痛點與複雜度，最後作者帶出自己認為好的架構及設計以及推薦不同的思路來處理
abbrlink: 41686
date: 2020-09-08 13:27:36
---

Article Translated from https://blog.container-solutions.com/gitops-the-bad-and-the-ugly, with author’s permission
本文翻譯自 https://blog.container-solutions.com/gitops-the-bad-and-the-ugly 已獲得作者同意

以下內容節錄自 [GitOps: The Bad and the Ugly](https://blog.container-solutions.com/gitops-the-bad-and-the-ugly) 並加上作者個人心得，有興趣瞭解
全文的人歡迎到上述網站去瞭解更多


# 前言
本篇文章非常有趣，作者本身是一個 GitOps 的擁護者，也有撰寫過相關的文章[11-reasons-for-adopting-gitops](https://blog.container-solutions.com/11-reasons-for-adopting-gitops) 來談論講為什麼要導入 GitOps,以及導入 GitOps 後能夠帶來的好處。

但是最近一些專案的經驗，讓作者開始感覺到一些 GitOps 所帶來的極限與痛點，加上最近與 [Humanitec](https://humanitec.com/) 這套並非基於 GitOps 的解決方案的開發人員交談後，作者決定撰寫本篇文章來探討自己於實戰經驗中所感受到的 GitOps 缺陷與極限，並於最後分享一些不同的思路及可能的解決方法


# 痛點



## 不適合使用於程式化的更新
如果今天 CI 流水線本身結束後，會需要將測試的結果送入到測試的叢集中去更新，那就意味者我們要透過修改 Git Repo 的內容來觸發 GitOps 的更新，因此需要在 CI 流水線的最後面對 Git Repo 進行修改。

然而 Git 這個工具的設計本身就不是很適合太自自動化的操作，特別是遇到衝突時，需要有人為的解讀來修復。所以當有多個 CI 流水線同時運作時，就會有機會導致有部分的 CI 流水線會遇到 Git 衝突導致沒有辦法順利完成作業。
> 這部分的衝突通常只是 Git 的紀錄更新，必須要一次又一次的 Pull 來抓到最新版才可以Push. 現實中很難真的發生會有單一檔案的衝突

為了解決這個問題，作者於他們的工作環境中又設計了其他的系統來不停重試這些環節，這過程花費大量的時間。

## Git Repo 增長帶來的問題

GitOps 強調一切都放在 Git Repo 上，因此隨者系統專案的增加， Git Repo 也會愈開愈多。而每一個 Repo 都需要進行相關設定，譬如權限，整合等

作者的親身經歷是他們有 30% 以上的時間來設定相關的 Git Repo。
作者認為減少 Git Repo 的數量可以減少這種痛苦，譬如每一個環境就用一個 Git Repo, 裡面放置該環境會用到的全部應用程式，但是這種架構下帶來的反而是一個 Repo 本身的權責太大，擁有的東西過多，然後搭配(1)的問題，就會產生更多衝突

## 缺乏視覺化
GitOps 強調一切都是 Git，所有的差異變化都可以透過 Git History 來追蹤，這也代表這些文字內容可以幫忙管理人員回答諸多部署的相關問題

然而作者的經驗是當環境愈來愈複雜時，有更多問題是沒有辦法依賴 GitOps 的特性來回答的，譬如說「某個應用程式被部署的頻率」。

作者認為 Git 內容的改動不太容易對應到特定應用程式的更動，舉例來說某些改動可能一次會觸發多個應用程式重新部署，而有些改動卻可能只有部分設定檔案改動，而沒有任何應用程式重新部署。

## Secret 的管理問題依然沒有解決
滿多的複雜應用環境都會遇到 Secret 的管理問題，特別是當 CI/CD 的流水線放置於系統外部服務時，這些機密資訊到底該如何保存?
舉例來說，凡舉資料庫的密碼或是私有鑰匙等需高度保存的東西都需要有一個很好的解決方案來保護。作者認為透過 Hashicorp 的 Vault 來進行集中化管理是一個比較合理的解法。

雖然 GitOps 本身沒有強調自己對於安全性管理有更好的效益與優勢，但是就實務經驗上來看，並沒有讓事情變得簡單。 畢竟 Git Repo 本身不是一個適合存放各種機密資訊的地方，就算透過加解密的方式只存放加密後的結果，這些過往的資訊還是會被 Git History 給永久保存。
此外當 Git Repo 的數量擴大變多時，這些加密後的機密資訊也就會散落愈多地方，會導致管理與追蹤這些機密資訊要花費更大的功夫來處理

## 缺少檔案資源的驗證性
作者認為 GitOps 的架構下, Git Repo 作為一個 Kubernetes 與 CI/CD 流水線中間的介面，沒有一個很好的方式去驗證所有修改過的資源是否有效與合法。
此架構下，則是開發者要確保自己的 Helm/Yaml 是否有格式問題，不然相關的修改被合併到 Git Repo 的話就會造成部署錯誤

我認為這個問題倒是不大，因為不管透過哪種部署思維都會有這樣的問題，反而是 CI/CD 的過程中能不能驗證 Yaml 的正確與否，如果可以提早驗證，並且阻止沒通過測試的程式碼被合併，那這些問題應該都不會發生


# 不同於 GitOps 的解決方案
上述的問題只是作者遇到的實戰問題，並非代表 GitOps 就毫無用處。 GitOps 依然帶來了很多好的優點，作者強調大家只需要心中知道你可能會遇到這些問題，然後遇到的時候可以想想要怎麼解決這些問題。

作者開始思考，什麼樣的解決方案可以保留 GitOps 的優點同事又可以增進上述的缺點，其心目中有一些基本要有的特性

1. 有能力記錄叢集環境上的一切變化
2. 使用宣告式(Declarative)的文件格式來描述或是設定環境上要用到的所有資源
3. 所有的環境變化都可支援審核機制，要通過審核才會往下運作
4. 權限控管，控制誰有能力去對環境資源進行更改
5. 有辦法針對 期望的狀態與運行的狀態進行比對

作者也認為一個好的架構中會有一個代理人運行在所有的 Kubernetes 之前，然後所有的操作請求都不應該直接面對 Kubernetes，而是跟該代理人溝通
如下圖
![](https://i.imgur.com/LF7SaTV.png)

該代理人本身提供 API 介面來接受各式各樣的請求，最後把這些請求更新到後方管理的 Kubernetes 叢集。

原文中還有更多關於這個架構的一些想法，有興趣的可以閱讀原文

作者也提到這部分會花很多的時間在設計這套解決方案，重新實作這個專案花的成本可能比專心用 GitOps 還來得高

剛好現在有一個專案 [Spinnaker](https://spinnaker.io/) 就是在實現上述的架構，其下一代開發 [Humanitec](https://humanitec.com/) 更是完全針對 Kubernetes 使用。
這些專案有上述架構提到的一些特性，有些還在開發藍圖，作者期望有一天他們能夠成為 GitOps 以外的選擇


# 個人心得:
作者提到的痛點我認為其實並不是每個使用 GitOps 的人都會遇到的，畢竟 GitOps 我認為是一個概念，底層要怎麼實作並沒有強烈限制。使用的開源軟體不同，最後的工作流程也會完全不同，因此不一定完全是 GitOps 的問題，更有可能是當前架構下的問題。

此外 Secret 的管理問題也並非是 GitOps 才會有的，而是任何在玩 CI/CD 的人都會遇到的問題，只是最後你要怎麼解決而已。

我也同意沒有最好的部署策略與解決方案，還是要根據自己的使用環境跟內部架構來挑選適合的工具

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
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>


# 參考來源
https://blog.container-solutions.com/gitops-the-bad-and-the-ugly
