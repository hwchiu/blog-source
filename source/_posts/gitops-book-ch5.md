---
title: '[書本導讀]- GitOps實作上的挑戰'
keywords: 'GitOps,DevOps,CI/CD,Kubernetes'
tags:
  - Kubernetes
  - Devops
  - GitOps
description: >-
  本文為電子書本[GitOps: What You Need to Know
  Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now)
  的心得第五篇。已獲得作者授權同意
abbrlink: 62675
date: 2020-10-04 11:50:33
---

本文大部分內容主要擷取自 [GitOps: What You Need to Know Now](https://info.container-solutions.com/gitops-what-you-need-to-know-now) ，已獲得作者授權同意

本文為 GitOps 系列文，主要探討 GitOps 的種種議題，從今生由來，說明介紹，工具使用到實作上的種種挑戰，讓大家可以從不同角度來學習 GitOps。

如果你想要嘗試將 GitOps 的概念導入到現有的團隊之中，這接下來我們會介紹一些實作是可能會遇到的問題。

# Technical
上篇文章中，我們探討了 GitOps 相關的工具選擇，然而這些工具發展快速，有些是基於開軟體，有些則是廠商的收費服務。廠商的服務通常都直接提供點到點的解決方案，所以如何選擇這些解決方案就是一個很困難的事情。對於已經團隊內已經運行很久的系統，通常都已經搭配不少工具來整合，這種情況下，如果要採用這些點到點全面的解決方案，勢必會有一些架構上的衝突。相反的，開源軟體很多都是獨立個體，彼此沒有太強烈的連結性，這種情況下也許會是一個更好的選擇。
當然如果今天要做的是一個全新的產品，沒有任何技術債的問題，那也許可以考慮點到點的解決方案。

當使用的工具選擇完畢之後，其實還有很多的技術問題要挑戰。譬如如何整合以及使用這些你選擇的工具，譬如 Secrets 這種機密資訊的管理，如果要整合到 GitOps 的環節中，該怎麼處理。

基於安全性為由，這些機密資訊絕對不能直接存放於任何 Git 專案下。否則任何能夠看到 Git 資訊的人都可以直接獲取你的機密資訊，為了解決這類型的問題，以下列出安全性相關的專案與類別。
## SealedSecrets
1. [Bitnami Implementation](https://github.com/bitnami-labs/sealed-secrets)

## Storing Encrpyted Secrets directly in your source reposiroty
1. [git-secret](https://github.com/sobolevn/git-secret)
2. [git-crypt](https://github.com/AGWA/git-crypt)
3. [BlackBox](https://github.com/StackExchange/blackbox)

## Storing secrts with source control separately from source
1. [GitLab protected variables](https://docs.gitlab.com/ee/ci/variables/#protect-a-custom-variable)
## Storing encrypted secrets with your source-control tool separately from source
1. [GitHub encrypted secrets](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)

## Storing secrets with your cloud vendor in a secrets-management system
1. [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
2. [Google Cloud Secets Manager])(https://cloud.google.com/secret-manager)
3. [Azure Key Vault](https://azure.microsoft.com/en-gb/services/key-vault/)

## Integrating with a third party secrets-management tool
1. [Hashicorp Vault](https://www.vaultproject.io/)
2. [Mozilla SOPS](https://github.com/mozilla/sops)

上述只是粗略的列出了跟安全性相關的專案，每個專案的用法與情境都截然不同。對於團隊來說，要花多少心力於這些安全性解決方案上取決於你們團隊對於安全性的重視度有多高及需求有什麼。
此外，最好定期檢視這些安全性的設定來確保一切設定都符合安全需求。

# Staffing
第二個問題與第一個問題息息相關，要找到一位對 Cloud Native 相關技術有經驗的人已經屬實困難，更何況要找到一位還要對 GitOps 熟悉有經驗的負責人。除此之外，要如何將這些概念擴散到整個團隊，讓團隊成員有相同的能力與背景共同處理這些流程，其難度更高。

一個比較可行的做法是透過學習的方式，讓員工之間花時間去學習分享與研究，藉此降低每個人之間對於 GitOps 認知的鴻溝，最後讓彼此都能夠掌握整個系統

# Regulatory and Legacy Tooling/Processes
GitOps 本身對於有監管需求的團隊來說是非常值得嘗試的，透過公開透明的檢查及瀏覽機制，可以讓團隊更能夠有效率地去知道什麼時候被修改，誰進行了修改，什麼內容被修改。

GitOps 下的做法相對簡單，只要將其與現有的技術與工具整合。舉例來說，如果今天團隊內使用 LDAP 或 AD 這種權限控管工具，可以很輕鬆的將其與 GitLab 進行整合。這樣就可以透過 LDAP/AD 來限制員工的權限，什麼群組的人可以觀看什麼 Repo，進行什麼操作。
這種概念的整合是個非常有效率的做法，特別是對大型的組織來說，能夠用這種整合的方式直接把現有的規則與政策都直接套用到新產品架構上，而不用重新打造一特全新作法。

然而，GitOps 的這些作法再某些領域上可能不會這麼順利，特別是跟些已經存在的工具有相反思維時。這邊就以 `Pull Request` 這種工作流程為範例，如果過去的開發習慣是手動硬上且一個帳號專門使用的系統，那就與 GitOps 的流程非常不和，因為沒有辦法做到稽核的效果。
另一方面，對於一些資深且不熟悉 Git 操作的管理人員來說，要其登入 Git 並且發送 Pull Request 可能會有些操作上的困難。

最後導致的就是，當你導入 GitOps 到團隊時，為了配合團隊舊有的工具或是稽核流程， GitOps 本身所強調的特性可能會被犧牲一些。 你甚至可能要花一些時間來研究現有機制，並且想辦法說服上層説為什麼採用全新的 GitOps 流程會是更好的解決方案。
這一切都沒有標準解答，完全是看各團隊到底習慣用什麼，怎麼用符合大家需求

# Time to Market
GitOps 方式帶來的好處非常容易透過白板解釋給技術背景的員工，但是要將其實踐並且整合則需要花費不少時間。 整個實踐過程需要仰賴非常紀律的方式去部署應用程式，手動介入的操作都要盡量避免，所有的測試都要寫好寫滿來確保工作流程。

事實上這種紀律的要求不是只有 GitOps 流派下才會需要，不論是測試驅動開發或是 DevOps 等都需要一定程度的紀律與準則來要求整個團隊。 這些紀律短期上可能看不出好處，但是其效益都是為了長期所打算的，當然這樣的做法帶來的缺點就是如果你要向上層展示其好處與優點，短期內可能很難展現。

上述提到的所有挑戰與困難都會增加團隊產品與市場接軌的時間，這可能會是 GitOps 實作上最大的挑戰。當面對一些真實市場的壓力與需求，團隊可能會傾向使用舊方法來處理產品部署的方式。

作者認為如果你沒有堅持下去，而是放棄使用過去的舊方法，那隨者時間久了，有一些競爭對手開始享受到 GitOps 帶來的長期效益時，這時候你的團隊在各方面就會追不上對方。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
