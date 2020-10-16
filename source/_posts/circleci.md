---
title: CircleCI 使用經驗談
keywords: 'DevOps,CI,CircleCI,CICD'
abbrlink: 28820
date: 2019-04-21 08:21:59
tags:
  - DevOps
  - CI/CD
description: 隨者愈來愈多的 CI/CD 工具被開發出來，對於維運/開發者來說，到底要選擇哪一套 CI/CD 工具來使用? 本文介紹了其中一款 SaaS 服務, CircleCI。 本文簡單介紹一下作者自己在選擇工具時會考慮的問題與情境，以及在什麼情境下選擇了使用 CircleCI, 最後介紹了幾個覺得好用的功能。
---

# Preface

2019/04/20 很榮幸有機會參加 [DevOps Taiwan - CI / CD / DevOps Pipeline Tools 大亂鬥](https://devops.kktix.cc/events/pipeline-tools-battle) 並且於該大亂鬥中跟大家分享我平常自己 Side Project 使用的一套 CI 工具, `CircleCI`.

本文的內容主要是基於該場大亂鬥中的[投影片](https://www.slideshare.net/hongweiqiu/introduction-to-circleci)進行更詳細的說明與範例

# Why CircleCI

在使用 `CircleCI` 之前，我也有使用過不少套類似的 `CI/CD` 的服務與工具，譬如 `TravisCI`, `Jenkins`, `Drone`, `Keel`, `Spinnaker`, `Argo` 等

其中工具有些是針對 `kubernetes` 去設計，有些則專注於 `CI`, 有些則是專注於 `CD`, 有的專注於 `Pipeline` 流程的設計。

對我來說目前也沒有看到任何一套能夠滿足所有需求的工具存在，每次選擇一套工具的時候腦中往往會浮現出下列各式各樣的問題

1. 預計使用該工具的人數會有多少? 有成本考量嗎?
    - 團隊內的人會需要操作嗎?
3. 團隊內成員的平均素質如何，能夠接受什麼樣的使用方式?
    - 有些人喜歡有簡單明瞭的 `UI` 操作，有些人喜歡什麼都要使用指令列操作，能不離開終端機最好。
4. 能否預估使用情境，評斷該工具的特色是否可以滿足需求?
    - 簡單使用的往往彈性不足，彈性足的往往設定以及維運複雜。
5. 該工具是否能夠與現存的其他工具整合?
    - VCS: Github/Github Enterprise/Bitbucket/GitLab/..etc
    - Jira/Trello/..etc
    - IDE?
6. 管理與維護是否方便? 若多人同時管理的情況下，有沒有辦法可以知道任何的設定跟動`是誰`造於`何時`造成的?
7. 該工具的維護團隊是否活躍？ 使用者生態系是否龐大? 有問題時是否容易尋找到解答 ?

<!--more-->
所以根據不同的使用情境以及不同的工作環境/團隊，上述的問題都會有不同的答案，最後會導出不同的選擇結果

而我自己在一些 `SideProject` 的使用中，我評估了上述的問題後則選擇了 `CircleCI` 來採用，原因如下

1. `SideProject` 基本上是我自己使用為主，所以我自己使用要舒服
2. 基於 `Yaml` 格式的設定，每個專案都會有一個屬於自己的設定檔案，意味可以透過 `版本控制` 的方式來控制該專案的 `CI` 流程，有任何跟動都可以透過 `版本控制` 的流程來追蹤
    - 個人不太喜歡只能透過 `UI` 設定的流程，因為團隊夠大的時候，往往會有人自行跑去修改了部分設定，年久失修後後會變成沒人知道為什麼會有這個設定，以及該設定的理由與原因
3. 所有的流程都是基於 `Yaml` 內的流程來設計，根據自己的需求去完成
4. 與 `Github` 整合
5. 提供 `SSH` 的方式提供偵錯方式，可減少無謂的錯誤的嘗試
6. 免費版本提供最多同時一個 `Job` 運行的使用量，對於 `SideProject` 的專案來說足夠了。
7. 採用 `CircleCI` 公司提供的 SaaS，本身不需要維護相關的機器，可專注於程式開發
8. `CI` 的運行環境支援 `Container` 與 `Virtual Machine`, 符合我開發上的需求。

基於上述的理由，我最後選擇了 `CircleCI` 來使用，接下來就來跟大家分享我使用上的一些心得


# Features
## SSH Debug
過往使用一些工具時，常常遇到設定出錯然後需要通靈的情況，因為也許該指令在自己的測試環境都可以正常運行，但是跑到 SaaS 服務上則不通，可能因為系統版本差異，軟體版本差異甚至是系統有些相關的 `Service` 本身沒開啟。

這種情況下要除錯都非常麻煩，幸好 `CircleCI` 提供了 `SSH Server` 的概念，讓開發者可以直接進去 `CI` 運行的環境進行操作，不但可以進行偵錯，也可以嘗試在該環境中將 `CI` 想要運行的指令都先行測試一遍，這樣要撰寫後續的設定檔就會相對輕鬆

於 Workflow 右上方可以重新運行該 Workflow, 並且開啟 SSH 的功能
![](https://i.imgur.com/eAf1aMN.png)

接者可以觀察到原先的 Jobs 中被加入了一個新的 `Enable SSH`，點開該 `Job` 即可得到連接資訊
![](https://i.imgur.com/4WFNFM9.png)

我本身的 `Github` 帳戶有設定 `SSH Key` 的綁定，因此將 `CircleCI` 與 `Github` 綁定整合後，創建的 `SSH` 環境會只能接受 `CircleCI` 綁定的私鑰進行登入

實際透過 `ssh指令` 登入到該環境後即可看到相關專案的內容，此時就可以在這個環境下去進行偵錯與測試。
![](https://i.imgur.com/lgZz9YJ.png)


## Executors
`CircleCI` 的運行環境中大部分都是基於 `Container` 來運行的，譬如下述的範例
```yaml=
jobs:
  build:
    docker:
      - image: circleci/golang:1.12

    steps:
      - checkout
      - run:
          name: Build Code
          command:  |
            sleep 10
            echo "Build Done"

  deploy:
    docker:
      - image: circleci/golang:1.12
```
這個範例內我會針對不同的 `Job` 去設定該 `Job` 要運行的環境，範例中都使用了官方提供的 `golang:1.12` 容器。

就如同寫程式一樣，相同的部分如果可以抽象一層對於後續的維護跟修改會相對的輕鬆一點，這一點上 `CircleCI` 則提供了 `Executor` 的語法讓開發者管理其設定檔案
```yaml=
executors:
  golang-ci:
    working_directory: ~/repo
    docker:
      - image: circleci/golang:1.12

jobs:
  build:
    executor: golang-ci
    steps:
      - checkout
      - run:
          name: Build Code
          command:  |
            sleep 10
            echo "Build Done"

  deploy:
    executor: golang-ci
```

這個範例中我們透過 `Executor` 的方式創造了一個名為 `golang-ci` 的運行環境，該環境中描述其使用的容器是 `circleci/golang:1.12`.
之後只要這些不同的 Job 會使用的環境都一致的話，可以繼續專注維護 `executors` 的部分即可。

此外由於所有的運行環境都是基於容器，所以其實可以自行客製化自行的運行環境，將必定會使用的工具與環境事先設定完畢，譬如透過 `pip3`, `apt-get`, `yarn`, `apk`, `dnf` 等各式各樣的工具來預先安裝需要的套件與工具，最後將這個環境打包成新的容器映像檔案。
最後再將該環境套用到自行的 `CircleCI` 運行環境中，就可以減少一些不必要的安裝動作，藉此減少每次運行所需要的時間。

# Commands
雖然 `Yaml` 本身已經有 [Anchors](https://medium.com/@kinghuang/docker-compose-anchors-aliases-extensions-a1e4105d70bd) 這種機制來避免重複撰寫設定的方式，然後 `Anchors` 裡面會用到大量的 `&`,`*` 這類型的符號，其實看了不是很順眼，同時 `Anchors` 是針對完全相同的設定來處理，若本身有參數的差異時就沒有辦法透過這個機制來處理。

`CircleCI` 提供了 `Commands` 的概念，可讓開發者將 `Jobs` 的描述方式撰寫成類似 `Function` 的概念，可設定呼叫所需要的參數，以及這些參數的型態與預設值。

下列的 `yaml` 則是一個簡單的範例，創建了兩個 `Commands`, 分別叫做 `setup_env` 以及 `deploy_env`.
這兩個 `Commands` 都有設定相關的參數以及對應的型態與預設值
最後可以透過 `CircleCI` 規範的使用方式 `<< parameters.$NAME >>` 的方式來存取。

```yaml=
commands:
  setup_env:
    parameters:
      environment:
        type: string
        default: "dev"
    steps:
      - run:
          name: "Setup the environment << parameters.environment >>"
          command: |
            echo  << parameters.environment >>
  deploy_env:
    parameters:
      environment:
        type: string
        default: "dev"
      seconds:
        type: string
        default: "5"
    steps:
      - run:
          name: "Deploy to environment << parameters.environment >>"
          command: |
            sleep << parameters.seconds >>
            echo  << parameters.environment >>
```

接下每個 Jobs 中就可以針對上述的 `Commands` 名稱直接使用，針對需要的部分可以傳遞參數覆蓋預設值，或是直接採用預設值即可

```yaml=
  deploy:
    executor: golang-ci
    steps:
      - checkout
      - setup_env
      - deploy_env

  deploy-staging:
    docker:
      - image: circleci/golang:1.12
    steps:
      - checkout
      - setup_env:
          environment: staging
      - deploy_env:
          environment: staging
          seconds: "10"
```

藉由 `Commands` 的方式能夠讓你好好的思緒有沒有辦法將你的 `Pipepline` 流程給設計的更模組化，將相同的部分抽出，透過不同變數的方式來減少重複撰寫的手續，同時也提高維護的方便性。

# Orbs
當有了 `Commands` 的概念後，就可以將滿多常見的功能給模組化，模組化的下一個步驟就是分享。這個概念其實與其他工具的 `Plugin` 雷同，只是在 `CircleCI` 中使用 `Orbs` 這個詞來表示。

可以到 [Orbs Registry](https://circleci.com/orbs/registry/) 看看目前提供的 `Orbs`

這邊我們以 [Slack Orbs](https://circleci.com/orbs/registry/orb/circleci/slack) 為範例，有自行撰寫過 `Slack` 相關通知的開發者就會知道在 `Bash` 中要透過 `Curl` 等方式去描述 `Json` 物件有多煩瑣。

我們可以看一下 `slack orbs` 裡面怎處理的
```yaml=
  - run:
      command: |
        # Provide error if no webhook is set and error. Otherwise continue
        if [ -z "<< parameters.webhook >>" ]; then
          echo "NO SLACK WEBHOOK SET"
          echo "Please input your SLACK_WEBHOOK value either in the settings for this project, or as a parameter for this orb."
          exit 1
        else
          # Webhook properly set.
          echo Notifying Slack Channel
          #Create Members string
          if [ -n "<< parameters.mentions >>" ]; then
            IFS="," read -ra SLACK_MEMBERS \<<< "<< parameters.mentions >>"
            for i in "${SLACK_MEMBERS[@]}"; do
              if [ $(echo ${i} | head -c 1) == "S" ]; then
                SLACK_MENTIONS="${SLACK_MENTIONS}<!subteam^${i}> "
              elif echo ${i} | grep -E "^(here|channel|everyone)$" > /dev/null; then
                SLACK_MENTIONS="${SLACK_MENTIONS}<!${i}> "
              else
                SLACK_MENTIONS="${SLACK_MENTIONS}<@${i}> "
              fi
            done
          fi
          curl -X POST -H 'Content-type: application/json' \
            --data \
            "{ \
              \"attachments\": [ \
                { \
                  \"fallback\": \"<< parameters.message >> - $CIRCLE_BUILD_URL\", \
                  \"text\": \"<< parameters.message >> $SLACK_MENTIONS\", \
                  \"author_name\": \"<< parameters.author_name >>\", \
                  \"author_link\": \"<< parameters.author_link >>\", \
                  \"title\": \"<< parameters.title >>\", \
                  \"title_link\": \"<< parameters.title_link >>\", \
                  \"footer\": \"<< parameters.footer >>\", \
                  \"ts\": \"<< parameters.ts >>\", \
                  \"fields\": [ \
                    <<# parameters.include_project_field >>
                    { \
                      \"title\": \"Project\", \
                      \"value\": \"$CIRCLE_PROJECT_REPONAME\", \
                      \"short\": true \
                    }, \
                    <</ parameters.include_project_field >>
                    <<# parameters.include_job_number_field >>
                    { \
                      \"title\": \"Job Number\", \
                      \"value\": \"$CIRCLE_BUILD_NUM\", \
                      \"short\": true \
                    } \
                    <</ parameters.include_job_number_field >>
                  ], \
                  \"actions\": [ \
                    <<# parameters.include_visit_job_action >>
                    { \
                      \"type\": \"button\", \
                      \"text\": \"Visit Job\", \
                      \"url\": \"$CIRCLE_BUILD_URL\" \
                    } \
                    <</ parameters.include_visit_job_action >>
                  ], \
                  \"color\": \"<< parameters.color >>\" \
                } \
              ] \
            }" << parameters.webhook >>
        fi
      name: Slack Notification
      shell: /bin/bash
```

可以看到滿滿的跳脫字元處理，整個使用與除錯想到就麻煩。

但是透過 `orbs` 的設計，我們可以重複使用上列的功能而不需要去管實作細節，整個使用範例如下

```yaml=
orbs:
  slack: circleci/slack@2.4.0

  deploy:
    executor: golang-ci
    environment:
      SLACK_WEBHOOK: https://hooks.slack.com/services/xxxxxxxxxxxxxxxxxxxxxxxx
    steps:
      - checkout
      - setup_env
      - deploy_env
      - slack/notify:
          message: Deploy dev done
```

根據 `Slack Orbs` 的文件解說，我們最需要的就是 `SLACK_WEBHOOK` 的設定，剩下都可以依據預設值來處理，這邊我針對訊息的部分進行了覆蓋。

透過這些語法與生態系，我們能夠讓整個設定檔更加的精簡與組織。


`CircleCI` 這部分發展的比較晚，目前提供的 `Orbs` 只有數十多個，為數不多，且目前沒有提供 `Private Orbs` 的使用。
這意味如果你有多個 Project 想要共用這些模組，你只能公開這些模組不然就是在每個 Project 的設定內去撰寫相同的 `Commands`。

# Cache
`CircleCI` 本身有提供 `Cache` 的機制，可以讓你在不同的 `Workflow` 中傳遞檔案與資料夾。 透過這個機制我們可以將一些中間產物給保存下來，譬如 `go build`, `npm build` 等建置後產生的檔案給共享，藉此降低下次執行相同 `Workflow` 所需要的時間。

使用上分成兩個概念
1. save_to_cache
2. restore_cahce

```yaml=
      - restore_cache:
          keys:
            - go-mod-v4-{{ checksum "go.sum" }}
      - run: make
      - save_cache:
          key: go-mod-v4-{{ checksum "go.sum" }}
          paths:
            - "/go/pkg/mod"
```

通常的使用邏輯都是產生這些中間產物後，透過 `save_cache` 的方式將特定路徑下的檔案給保存起來。同時我們需要針對這份 `cache` 給一個名稱。
使用上都會搭配 `checksum` 的方式來幫這份 `cache` 上一個跟其內容有關的名稱。

這樣下次運行的時候，透過 `restore_cache` 搭配特定的 `key` 就可以從眾多的 `cache` 中取出該資料並且覆蓋回去該系統中。

上述的範例可以這樣解讀，只要 `go.sum` 這個描述中間產物來源的檔案本身沒有跟動，那我們就可以放心的繼續採用相同的 `cache`. 但是一旦本次的更動有改變到 `go.sum`，不論是新的 library 或是版本的更新，都會導致 `go.sum` checksum 的改變，所以就會要到一個不存在的 `cache`. 最後就會迫使 `workflow` 去產生一份全新的 `cache` 供未來使用。


# Summary
除了上述的功能外，還有一些滿多內建功能可以使用，譬如 `store_artifacts`, `store_test_results` 等，這部分就是有需要的時候再到官方文件查詢即可。

再次重申，沒有最完美的工具能夠適合各種情境，只有想清楚自己的需求與情境，在針對需求去選擇適合自己的工具。此外需求確定的情況下來尋找合適的工具也可以避免踩入一套工具最後卻很難抽身的情況。

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

# Reference
- [CircleCI](https://circleci.com/docs/)
