---
title: '[書本導讀]-什麼是GitOps'
keywords: 'GitOps,DevOps,Kubernetes'
tags:
  - Kubernetes
  - Devops
  - GitOps
description: >-
  本文為電子書本[GitOps: What You Need to Know
  Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now)
  的心得第二篇。已獲得作者授權同意
abbrlink: 14649
date: 2020-09-23 19:26:43
---

本文大部分內容主要擷取自 [GitOps: What You Need to Know Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now) ，已獲得作者授權同意
本文為 GitOps 系列文，主要探討 GitOps 的種種議題，從今生由來，說明介紹，工具使用到實作上的種種挑戰，讓大家可以從不同角度來學習 GitOps。

# 前言

如同前篇文章所述， GitOps 這個概念是由 `Weaveworks` 於2017所發表的文章中提出，以下是該篇文章中的結論

GitOps 可以用兩件事情來總結
1. GitOps 是一種維運模型，專門用來針對 Kubernetes 以及其他 Cloud Native 技術。GitOps 提供了一種專門針對容器化叢集及應用程式的最佳實踐，其統一了部署，管理以及的監控等技術。
2. 提供一條給開發者更好的管理應用程式道路，End to End 的 CI/CD 流水線以及 Git 工作流程可以同時應用到開發與維運上


上述兩概念可以用來解釋 GitOps 的精神，但是其實並沒有解釋得太清楚到底什麼是 GitOps 及為什麼其帶來的改變這麼大。
所以接下來就會針對 GitOps 來探討其本質

# The Core of GitOps
GitOps 有三個核心價值，等下我們會針對這三個核心價值詳細介紹
1. Audited Change Management of Source Code
2. Declarative Data Definition of Systems
3. Control Loop Configuration Management of Systems

這三個核心價值一起應用時，可以提升
1. 可靠性，透過
    a. 自動化且 zero-touch (自動化設定，不需人為介入控制)的系統
    b. 更少對系統造成不被監控的修改
    c. 宣告性的作法，而不是透過程式化去產生
3. 稽核性以及當責性
    a. 透過來源控制工具來檢視完整的修改紀錄


除了這兩個特性外，還可以帶來這些優勢
1. 提高生產性以及降低管理所需要的工作成本
    a. 可以花更少時間來除錯那些沒人知道的狀態
    b. 可以花更少時間來維護那些不停發生的系統問題
    c. 測試可以更早發現問題
    d. 搭建簡單且低成本的工作流系統
3. 提高測試性
    a. 透過 pipeline 系統的自動化可以達成更輕鬆與自然的達成自動化測試

# Breaking Down the Core
接下來針對這三個核心價值進行更深一步的探討
## Audited Change Management of Source Code
當程式碼改變後，現代的原始碼管理工具必須要能儲存全部的管理資訊，包含誰修改了資訊，做了什麼修改，以及誰准許這次修改

為了達到這些功能，這工具必須要可以提供
1. 對於程式碼能夠簡單的達到分支與複製的功能
2. 透過 hashing 的方式來確保歷史修改紀錄的唯一性
3. 支持 Pull Request 類似的功能
4. 跟一些身份認證系統的整合，譬如 LDAP

就目前而言，只有一套工具可以被考慮來滿足上列的要求，這也是為什麼`Git`這個詞會在這個概念中被強調，直接用 `GitOps` 來描述一切。
但是這邊要特別注意，除了 Git 之外還是有其他工具有潛力可以滿足上述之些條件，最重要的就是能夠滿足 GitOps 的概念即可。
> 目前沒有一個很強勁的競爭對手於原始碼控制市場中可以與 Git 匹敵

## Declarative Data Definition of Systems
整個概念的原則就是不論是軟體或是架構的部署，目標都是透過原始碼內的檔案來定義，藉由這些檔案來定義其最終的期望狀態，但是到底這些期望狀態怎麼達到，則不是這些檔案要關注的。

最知名的一個關於這個概念的範例就是 Kubernetes 系統中的 YAML 檔案， Kubernetes 透過 Yaml 來定義到底什麼樣的資源要被部署到叢集中，或是什麼樣的資訊應該要被儲存在叢集中來輔助其他的應用程式。 其他的工具譬如 Pupper 都有類似的方式。

舉例來說，下圖是一個 YAML 的範例，其中就定義了 Secret 這個資訊，並且將其掛載到
Kubernetes 內的容器應用程式。這邊要注意的是下述的範例並沒有定義到底 Secret 要怎麼掛載，或是定義檔案內不同的資源被處理的先後順序。
透過原始碼內來宣告`宣告`一個理想的系統，最後怎麼部署跟處理的則是由其他的工具處理

![](https://i.imgur.com/yI8Zzvs.png)

相對於其他不同的部署方式，不論是明確定義部署先後順序，或是將定義狀態與實際部署邏輯混雜一起處理。 GitOps 這種將狀態與部署邏輯分離的方式可以更簡單與輕鬆的去管理系統。

## Control Loop Configuration Management of Systems
`Control Loop` 是一個不停運行的程式，運作邏輯滿簡單
1. 檢查當前目標系統內的運行狀態跟期望狀態是否一致
2. 如果不一致，則將系統內的運行狀態改成跟期望狀態一致
![](https://i.imgur.com/InccrTF.png)

一個經典的範例就是家中的空調系統，其不停地監控當前的溫度並且確認設定的溫度是否一致，如果不夠冷就讓他更冷點，直到兩邊的溫度一致。
Control loop 在大型的分散式架構系統中是一個很強力的部署應用程式工具，透過這種簡單的演算法就可以將應用程式同時部署到多個叢集中。

# Key GitOps Tools
先前已經提過 Git 是實現整個 GitOps 的核心的重要工具，接下來會列出一些跟 GitOps 有關的工具。

## Kubernetes
Kubernetes(K8s) 是一個開源的容器管理平台，旨在提供自動化的應用程式部署，縮放與管理。
Kubernetes 自從 Google 2014 開源以來就發展迅速，基本上 Kubernetes 在其領域的統治力就跟 Git 差不多強烈。

K8S 主要是使用 YAML 作為其宣告式語言的基礎，透過 YAML 來定義各式資源的期望狀態。

此外， Kubernetes 內部也有 Control Loops 的核心概念，叢集中的每個節點上面也都會運行一個 Control Loops 來確保每一台節點上應該要運行什麼諮詢，不應該要運行什麼資源。


## Terraform
Terraform 是一個開源的 `Infrastructure as code`(架構即程式碼) 的開源工具，Terraform 的設定檔案都是基於 Hashicorp 所開發的 HCL 這種宣告式語言，藉由這種語言使用者可以用來定義其架構中的期望狀態。
其他的雲端廠商也有提供類似的工具來創建其基礎架構，譬如 CloudFormation 這種工具，不過 Terraform 則是透過 plugin 系統來支援更多的廠商。

## Docker/Containers
容器這個技術發展已久，甚至在 Docker 之前就已經存在與發展，但是自從 2013 年 Docker 發展後，Docker 成為一個非常強勁的容器解決方案。 透過 Docker 的技術，使用者可以很輕鬆的將其應用程式給打包起來並且將其運行到各式各樣的環境之中。

# Summary
結論來說， `Gitops` 的核心概念基於三個概念
1. 透過宣告式定義應用程式的期望狀態
2. 透過原始碼控制來管理整個修改歷史
3. Control loop 的概念來管理整個應用程式的交付

這些概念一直以來都不是全新的發想，但是因為 GitOps 常用的工具們(Git, Docker, Kubernetes 以及 Terraform)自從 2015 後都逐漸發展並且壯大出名。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
