---
title: '[書本導讀]-GitOps工具的選擇'
keywords: 'GitOps,DevOps,CI/CD,Kubernetes'
tags:
  - Kubernetes
  - Devops
  - GitOps
description: >-
  本文為電子書本[GitOps: What You Need to Know
  Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now)
  的心得第四篇。已獲得作者授權同意
abbrlink: 32349
date: 2020-09-30 09:42:46
---

本文大部分內容主要擷取自 [GitOps: What You Need to Know Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now) ，已獲得作者授權同意

本文為 GitOps 系列文，主要探討 GitOps 的種種議題，從今生由來，說明介紹，工具使用到實作上的種種挑戰，讓大家可以從不同角度來學習 GitOps。


GitOps 的工具選擇非常多，市場上的發展非常快速，本篇文章就針對不同的作法介紹一些可用工具
> 這邊介紹的是參考工具，並非是唯一選項

# GitOps Tooling Categories
## 'Push' GitOps Tools
`Push` GitOps 的工具(技術上來說，可以稱為基於 Client-API 的工具)也許可以被視為相對老舊的工具。本質上來說，其實作 `Control Loop` 的概念於 CI/CD 流水線系統上，在 CI/CD 的過程中，去管理系統上的應用程式狀態，並且將差異性給部署到系統中。
這種架構中，這個 CI/CD 的工作可以有很多種被觸發的方式，譬如是排程的觸發(每天晚上)，或是任何程式碼更動導致的觸發(PR,Merge)。

一個比較容易理解範例是，假設有一個公司使用 Jenkins 作為其 CI/CD 流水線系統來處理與 Kubernetes 上的應用程式。當流水線被觸發後，程式碼中那些宣告式的檔案就會被重新部署到 Kubernetes 叢集內來想辦法達到期望的狀態。
這部分可以借助一些指令列工具，譬如 `kubectl`, `kustomized` 等。
此外如 [kubestack](https://www.kubestack.com/) 這類型的工具也是基於相同的策略來完成，不過實作則是透過 Terraform 等工具來完成 Kubernetes 內的部署。

最近這幾年，一些Git管理服務譬如 GitLab 以及 GitHub 都發展了屬於自己的一套 CI/CD 流水線，如 [Github Action](https://github.com/features/actions) 以及 [GitLab CI/CD](https://docs.gitlab.com/ee/ci/)，其架構中提供了豐富的第三方套件系統讓其系統能夠提供更廣泛與強大的功能。

對於設定檔管理工具來說， Ansible 可以被視為是一個 `push` GitOps 工具，其底層實作是基於 SSH 連線來將所有的修改都套用到遠方環境。

`Push` GitOps 的工具的特性使得開發團隊能夠用相對簡單的概念去理解與維護整個部署流程。然而，隨者時間發展，有一些團隊開始因為這個流程對於 `Control Loop` 的無力感而感到灰心。

無力感的理由如下
1. 對一個已經存在的資源進行更新時，如果本次的修改有包含一些資源的刪除，那要將遠方系統上這些相對應的資源也一併刪除是一個實作上的挑戰
2. 通常來說，將修改內容套用到系統上通常會需要一系列的腳本幫忙輔助，而這些腳本彼此運行的順序尤其重要，一但順序錯誤可能整個邏輯就大亂，最後部署的結果也會不如預期。


## 'Pull' GitOps Tools
'Pull' GitOps 工具(技術上來說可以說是基於代理人架構)完全採用不同的方式，這些工具會先於目標環境中部署一個代理人服務，而該代理人服務則會等待各種狀態變動的請求，並且根據這些請求來修正系統上的狀態。
如果是設定檔管理工具的話， Puppet 則是一個基於 Pull 機制的管理工具，所有的更動都要環境中的代理人幫忙管理

使用這種模式的好處就是這些代理人因為本身就處在這些目標環境中，所以可以被視為是一個可信任的代理人。以 Puppet 來說，其代理人可以基於 `privileged` 這種比較有權限的使用者去運行。

另外，對於 Kubernetes 來說，內部的 Operator 概念就是完全符合這種模式，透過授權的代理人能夠存取 Kubernetes API 來獲得當前系統上資源的狀態，甚至對其修改。

相關的工具譬如 [Flux/ArgoCD](https://blog.container-solutions.com/fluxcd-argocd-or-jenkins-x-which-is-the-right-gitops-tool-for-you) 就會於 Kubernetes 內運行一個代理人，透過適當的設定後，這些代理人就會專注於特定的 Git Repo，當有任何更動時就會把差異內容套用到 Kubernetes 內，甚至有必要的話還可以把系統上的更動寫回到 Git Repo 內確保狀態一致。

其他工具譬如 [Tekton](https://cloud.google.com/tekton) 則是一套基於 Kubernetes 內的 CI/CD 工具，能夠輕鬆地做到上述 GitHub Action 或 GitLab CI/CD 與kubectl結合的各種事項。


## Infrastructure Provisioning Tools
對於基礎建設來說，GitOps 領域中最成熟的使用工具則是 Terraform。其能夠成為這個領域的佼佼者主要依賴於其乾淨且簡單的宣告式語法，讓使用者可以順利的專注於基礎架構的建置。

相對於 Terrafrom 來說，不同的公有雲廠商都有推出針對自己環境的工具，譬如 AWS 的 Cloud Formation, Azure 的 Azure Automation 或是 Goolge 的 Deployment Manager。
這些工具都專注於其開發廠商，然而 Terraform 則透過套件系統讓其能夠支援的環境非常多樣化且具有發展性。

最後，Kuberentes 本身則是透過 ClusterAPI 這個專案來達成 GitOps 版本的基礎架構交付工作，透過這個專案我們可以用 Kubernetes 內習慣的宣告式語言來定義 Kuberentes Cluster 並創建。
過往的狀態下，使用者都是基於各家廠商自有的 API 去開發相關的工具，更多情況下則是手動操作。
但是透過 ClusterAPI 的發展，我們可以利用 Kubernetes 本身的機制來幫我們創建與管理其他的 Kubernetes 叢集，只要透過 ClusterAPI 去規範與定義目標狀態即可。


## Curated GitOps Products
因為 DevOps 這個領域實在太深太廣，工具的選擇完全沒有結束的一天，大家總是針對不同的環境下，對不同的工具進行優缺點的比較。基於此概念下，愈來愈多全新的開放原始碼軟體就被發展了，其中最重要的就是 Jenkins X.

JenkinsX 其目的並不是要取代 Jenkins，其本身運行在 Kubernetes 叢集內，透過與其他工具譬如 [Skaffold](https://skaffold.dev/),[Tekton](https://github.com/tektoncd/pipeline),[Lighthouse](https://github.com/jenkins-x/lighthouse), [Kaniko](https://cloud.google.com/blog/products/gcp/introducing-kaniko-build-container-images-in-kubernetes-and-google-container-builder-even-without-root-access) 的整合來打造一個基於 GitOps 架構的 Cloud Native CI/CD 工具。

過去數年來， GitLab 也發展了不少的工具來強化他們本身與 Kubernetes 的整合，提供更多針對容器環境的自動化的測試與部署。 藉由 Gitlab Autodevops 的幫忙，你可以享受到程式碼的安全性掃描，容器的安全性掃描，動態以及靜態的應用程式安全性測試，不同的部署策略以及更多。


# GitOps Tooling Summary
以下是上述工具的總結

---------
GitOps 'Push' Tools
1. GitLab CI/CD + Kubectl/Helm
2. GitHub Actions + Kubectl/Helm
3. Kubestack

---------
GitOps 'Pull' Tools
1. ArgoCD
2. Flux

---------
GitOps Infrastructure Provisioning Tools
1. Terraform
2. Pulumi

---------
Curated GitOps Productis:
1. Jenkins X

這章節就是很粗略的介紹目前 GitOps 中的一些可用工具，當然實際上有各式各樣的工具可以使用，同時也會有更多新的工具被發展出來，解決出不同的問題。此外工具的整合也是進行式，特別是 ArgoCD 與 Flux 於 2019 宣布要加入此專案[Argo Flux](https://www.weave.works/blog/argo-flux-join-forces)，後續的發展指日可待。

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



