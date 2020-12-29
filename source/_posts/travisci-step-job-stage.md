---
title: '[DevOps] Travis CI - Step/Job/Stage'
keywords: 'GitHub,CI/CD,Travis,TravisCI, Travis CI'
abbrlink: 55114
date: 2018-09-01 09:09:13
tags:
  - CI/CD
  - DevOps
  - Linux
description: 這次要跟大家分享的是一些關於 TravisCI 的使用心得，相信有在 Github 上面維護專案的人應該都對各式各樣的 CI 系統不陌生，不論是 公有服務的 TravisCI 或是 CircleCI 或是自己透過 Jenkins 來處理。本篇想要跟大家分享的重點是在 TravisCI 上關於 Build Stage 的概念，透過 Build Stage，我們可以更有架構的去設計該專案的 CI/CD 流程。

---

# Preface

舉例來說，假設我的 `Github` 專案會希望每次`Release`時會有下列的行為
1. 進行 單元測試/整合測試 等各種確保程式碼正常運作的測試
2. 建置 `Docker` 相關映像檔並且更新到相關的容器倉庫
3. 將最新的程式碼部屬到相關的環境上

接下來會跟大家分享一下，在 `TravisCI` 的設定中，我們有什麼辦法可以滿足上述的需求

本文最後用到的 `travisCI` 設定檔可以在 [TravisCI Example](https://github.com/hwchiu/Travis-MultisStage/blob/master/.travis.yml) 找到


# Build Steps
所謂的「工欲善其事，必先利其器」，在使用 `TravisCI` 來解決我們的問題之前，我們必須要先瞭解 `TravisCI` 的基本概念，然後思考如何用這些基本概念來完成我們的需求。


## install/script
整個 `TravisCI` 的生命週期是由兩個主要步驟來構成
1. install: 用來安裝任何相依性套件的階段
2. script: 真正用來進行 `CI` 相關測試的階段


除了上述這兩個步驟外，也有所謂類似 `Environment` 這種`非必要`選項可以讓整個 `TrasivCI`設定更簡潔
譬如可以讓 `TravisCI` 幫你準備相關的環境程式語言環境與特定版本，譬如說 `golang 1.8, 1.9`

對於 `install/script` 這兩個執行步驟來說，本身為了讓運作邏輯更加細膩，所以又衍生出了前後步驟的概念
1. before_install: 該步驟會在 `install` 前運行，主要是用來準備任何 `install` 步驟所需要的資源，譬如透過 `apt-get update` 等更新套件倉庫。
2. before_script: 該步驟會在 `script` 前運行，如同 `before_install` 一樣，為了 `script` 進行資源準備來滿足真正測試所需，譬如資料庫的建置
3. after_script: 當 `script` 運行完畢後會執行的步驟，實際上還有所謂的`after_success` 以及 `after_failure` 更細部的針對測試的結果來區分的步驟。


根據相關人員在 `Github Issue` 的回答， `before_install` 以及 `before_script` 的使用時機如下

**before_install** runs before the install step, which is meant to install any required packages or dependencies. You can prepare things before you run this step, or you can e.g. run sudo apt-get update to refresh the apt indexes.

**before_script** runs before the actual test/build script runs. It's commonly used to run any preparation steps required to get the build running, for instance copy database configurations, set up any additional environment configuration, and so on.



## Example
以下示範一個非常簡單的 `.travis.yml` 設定檔案，在此環境中，我們要求 `TravisCI` 準備一個 `golang 1.8` 版本的環境，同時對於 `install` 以及 `script` 這兩個階段我們都執行非常簡單的指令。

```yaml=
language: go

go:
  - "1.8"

before_install:
  - echo "before_install"

install:
  - echo "install"
  -
before_script:
  - echo "before_script"

script:
  - echo "script"
```

上述的運行結果如下圖，該圖示我們可以觀察到
1. golang 版本 1.8
2. 四個步驟的結果依序輸出

![Imgur](https://i.imgur.com/14TXbFV.png)


## Solution
有了關於 `TravisCI` 建置週期的概念後，回過頭來探討一些下最初的需求

1. 進行 單元測試/整合測試 等各種確保程式碼正常運作的測試
2. 建置 `Docker` 相關映像檔並且更新到相關的容器倉庫
3. 將最新的程式碼部屬到相關的環境上

首先，這三個要求是有依賴性的，前面的失敗，後面的就不需要運行。
這邊沒有一個標準答案，有非常多的實現方式，譬如說
1. 將所有的步驟都放在 `script` 這個步驟去依序執行
2. 使用 `script`, `after_script` 甚至是其他 `deploy` 等不同的步驟來依序完成這些事情

譬如下面範例 (單純舉例)

```yaml=
language: go
go:
  - "1.8"
before_install:
  - echo "before_install"
install:
  - echo "install"
before_script:
  - echo "before_script"
script:
  - go test -v ./...
after_script:
  - sudo docker build -t ....
  - sudo docker push ....
deploy:
  - ./deploy.sh

```

可以到官方網頁這邊學到更多不同的建置步驟以及彼此之間的先後關係
https://docs.travis-ci.com/user/customizing-the-build/

用下列這張圖來幫這個章節做一個總結

![Imgur](https://i.imgur.com/Mrb22oE.png)

# Job
## Definition
瞭解了基本的用法後，我們要來看看一些關於 `TravisCI` 的進階用法，看看透過這些進階用法我們能夠完成什麼樣更豐富的 `CI` 流程。

首先，我們先定義什麼叫做 `Job`, `Job` 就是一個歷經 `TravisCI` 生命週期所有步驟的基本單位。

所以一個`Job` 簡單來說會經歷過 `Environment`, `before_install`, `install`, `before_script`, `script` 以及 `after_script` 所有步驟

這邊列舉的步驟並不是`TravisCI`的所有步驟，只是舉出幾個常見的步驟


## Multiple Job
有了 `Job` 的基本概念後，我們就可以往下思考一些更進階的用法。

假設專案本身是透過 `golang` 程式語言撰寫而成的，我們現在希望測試該專案在不同 `golang` 版本下是否都能夠正常運行。
舉例來說，我希望使用 `golang 1.8` 以及 `golang 1.9` 這兩個版本來進行專案的測試。

問題來了，這種情況下，我們要如何透過`TravisCI`來完成呢?
最直覺的就是我們什麼都硬幹，自己在 `TravisCI` 內去安裝各式各樣的環境，然後撰寫腳本去分開各式各樣的測試，將所有的需求都在**一個Job** 內完成。
當然這種情況下整個環境準備/測試等相關的邏輯就會複雜且不好維護

為了讓整個測試的架構乾淨與明瞭，我們可以透過 `Travis` 平行的運行多個`Job` 來滿足我們的需求。
在此架構下， `Travis` 會併行的去運行這些 `Job`, 且每個 `Job` 都有自己的建置週期，所有的 `Job` 都要都要成功該次測試才算成功。

使用下列圖示再次說明 `Multiple Job` 的概念
![Imgur](https://i.imgur.com/1knugMM.png)

上方描述的是一個簡單的 `Jobs` 概念，涵蓋了本文提及的基本建置週期。
下面則是為了滿足特別需求，希望多個`golang` 版本同時測試，此時我們就可以一次運行多個 `Job`, 其中只有 `Environemnt` 的部份是完全獨立，其餘則是都會採用相同的設定。

在 `TravisCI` 的設定檔案 **.travis.yml** 裡面，我們可以用下列的方式完成這個需求
```yaml=
language: go

go:
  - "1.8"
  - "1.9"
before_install:
  - echo "before_install"
install:
  - echo "install"
before_script:
  - echo "before_script"
script:
  - echo "script"
```

最後運行的結果如下, 可以看到該次的測試同時運行了兩個 `Job`, 這兩個 `Job` 分別是不同的 `Golang` 版本。
![Imgur](https://i.imgur.com/sr2xvlX.png)

## Custom Job
上述我們利用個 `go` 這個由 `TravisCI` 所定義的語法來完成產生 `MultipleJob` 的功能。

這時候腦筋一轉，`install`,`script` 這些建置步驟是否也都可以有類似的概念呢?

舉例來說，我希望對我的專案進行不同的測試，譬如 `Unit Test`, `Integration test`。
而這些測試除了測試的方式不同之外，環境的準備也不同
此外，同時運行這些測試也能夠減少測試的時間，並且將測試結果更清楚的標示出是哪種測試出問題。

將上述的需求轉換成 `TravisCI` 的概念的話
就是需要同時運行多個 `Job`, 每個 `Job` 裡面對於每個建置步驟都有自己客製化的需求。

這個需求我們透過下圖視覺化的方式來重新檢視一次

![Imgur](https://i.imgur.com/Kr9KDi3.png)

我們的需求很簡單，希望同時運行多個 `Job` 且這些 `Job` 針對不同的運行階段能夠選擇是否要使用預設的規則或是客製化自身的需求。


`TravisCI` 就有提供了這樣的功能供各位去使用，在其 `.travis.yml` 這個 `yml` 的檔案中，我們要透過 `jobs:include` 的概念去撰寫


```yaml=
language: go

go:
  - "1.8"

before_install:
  - echo "before_install"
install:
  - echo "install"
before_script:
  - echo "before_script"
script:
  - echo "script"

jobs:
  include:
    - stage: Custom Testing
      name: Unit-Testing
      go: "1.8"
      script: echo "unit script"
    - name: Integration-Testing
      before_install: "Integration-Testing_before_install"
      go: "1.9"
      script: "Integration-Testing_script"
```

其運行結果如下
![Imgur](https://i.imgur.com/PhjZheD.png)

我們可以觀察到我們的確運行了兩個 `Job`  而這兩個 `Job` 都有明確的名稱，這邊就沒有點進去看各自 `Job` 的運行結果，有興趣的人可以自行嘗試看看。


這邊先不討論語法，等到所有的概念都講述完畢後，再來討論語法的撰寫。


# Build Stage

有了上述的 `Multiple Job` 的概念後，我們重新審視一下最初的需求
1. 進行 單元測試/整合測試 等各種確保程式碼正常運作的測試
2. 建置 `Docker` 相關映像檔並且更新到相關的容器倉庫
3. 將最新的程式碼部屬到相關的環境上

首先，不同的測試本身可以透過同時運行多個 `Job` 來滿足，這邊好處理。

那建置/更新 `Docker Image` 這件事情，我們要讓誰來處理?
1. 上述的測試選一個 `Job`,  客製化其某些建置步驟來處理
2. 額外產生一個 `Job` 來專門處理這類的需求

只是採用第一種方式可能會有一個問題
假設我們需要 `所有` 測試都通過才能進行 `Docker` 相關的處理，那我們就沒有辦法在任意一個測試 `Job` 內去處理這個邏輯。

為了解決這個問題，除了將所有的工作重新集中回一個`Job`處理外，就只能在開啟第三個 `Job` 來處理。

但是這個 `Job` 本身有相依性的問題，它必須要確認前述相關測試的所有 `Job` 都完成才能夠繼續往下運行。
為了解決這個問題，我們要在這邊介紹 `Stage` 這個概念。

`Stage` 的特色以及概念如下
1. 由一群 `Job` 組成
2. 只要有一個 `Job` 失敗，該 `Stage` 就會被視為失敗
3. 只有當該前 Stage 是成功的狀態，才會執行下一個 Stage

有了 `Stage` 的概念，我們可以把上述的需求重新整理歸納成三個 `Stage`

- Testing Stage
    - Unit Testing Job
    - Integration Testing Job
- Docker Build Stage
- Deploy Stage

將這個概念用下圖再次檢視一次

![Imgur](https://i.imgur.com/YkGmTQN.png)


## Example
每個 `Stage` 之間彼此有依賴性，只要當其中一個 `Stage` 失敗，就不會往下執行

下圖是每個 `Stage` 都順利執行的成果
![Imgur](https://i.imgur.com/nm3COBH.png)


下圖則是當第一個 `Testing Stage` 有任何失敗的結果
![Imgur](https://i.imgur.com/pyyloCh.png)


```go= .travis.yml https://github.com/hwchiu/Travis-MultisStage/blob/master/.travis.yml .travis.yml

language: go

go:
  - "1.8"

before_install:
  - echo "before_install"
install:
  - echo "install"

before_script:
  - echo "before_script"
script:
  - echo "script"

jobs:
  include:
    - stage: Custom Testing
      name: Unit-Testing
      script: echo "unit script"
    - name: Integration-Testing
      before_install: echo "Integration-Testing_before_install"
    - stage: Build Docker Image
      script: echo "docker build"
    - stage: Deploy
      script: echo "release"
```

首先，我們要先定義 `Stage`, 在 `Stage` 裡面可以定義多個 `Job`, 而每個 `Job` 內又可以自定義每個建置階段，若沒有特別設定的，都會採用最上層的全域設定

定義 `Stage` 則採用 `stage` 這個關鍵字來幫建立，並且命名，針對每個 `job` 可以透過 `name` 的方式把對應的名稱替換掉讓整個測試報告更有閱讀性，然後接下來就可以去描述每個建置步驟，如 `script`, `install` 等各式各樣的建置週期步驟。

更詳細的設定可以直接參考[官網的說明](https://docs.travis-ci.com/user/build-stages#how-do-build-stages-work)


# Summary
本文跟大家分享了關於 `TravisCI` 的使用心得，從基本的使用方法到進階的 `Multiple Job` 以及 `Stage`

透過這些概念的組合，我們能夠將 `CI/CD` 的流程拆的更細緻，讓整個架構與流程更加清楚，同時透過平行運行的方式加快整體流程的速度 **(這部份不一定，完全是看每個專案的流程)**.

## Reference
- https://github.com/travis-ci/travis-ci/issues/1392
- https://docs.travis-ci.com/user/customizing-the-build/
- https://docs.travis-ci.com/user/build-stages

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
