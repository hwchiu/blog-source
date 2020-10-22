---
title: Management AWS Profiles
keywords: 'AWS, Shell, zsh, on-my-zsh'
abbrlink: 57841
date: 2019-06-08 02:49:03
tags:
  - AWS
  - Shell
description: 本文分享如何透過一些常見的方法或是別人撰寫好的工具來提供一個方便的管理工具，讓操作者可以更方便的再多個 AWS 帳號中進行切換

---

# Preface
如果有想要透過 Command Line Interface 來管理 AWS 帳號的人，應該都會有參考這篇官方的[Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) 來設定相關的工具與環境。

這邊簡單做個快速的概括，想要在 Shell 上面操作 AWS 帳戶的話，要執行下列的動作
1. 下載並且安裝 awscli (可以透過 pip 安裝)
2. 設定 AWS Profile


對於一般使用 `AWS` 通常到這一個步驟就差不多完成了，能夠將帳戶給設定完成並且使用 `aws` 相關的指令來存取公有雲上的資源。

然而對於部分企業或是使用者來說，基於商業/安全/其他考量，可能會有多組 `AWS` 的帳號，要如何在 Shell 中方便且快速地切換不同的 `AWS` 帳號則是一個有趣的問題。
接下來會跟大家分享我自己是如何管理多組 `AWS` 帳號的

# Introduction
再開始探討如何切換多組 `AWS` 帳號前，我們要先瞭解 `aws cli` 本身是怎麼運作的。

`aws cli` 本身有兩個檔案用來處理帳號的基本設定以及相關的認證資訊, 分別是
`$HOME/.aws/config` 以及 `$HOME/.aws/credential`

其中 `$HOME/.aws/config`  裡面的格式大概是
```bash=
[default]
region = us-west-1
output = json

[profile Account1]
region = us-west-1
output = json

[profile Account2]
region = us-west-1
output = json
```

```bash=
[default]
aws_access_key_id = xxxxxxxxxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxx

[Account1]
aws_access_key_id = xxxxxxxxxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxx


[Account2]
aws_access_key_id = xxxxxxxxxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxx
```

當執行 `aws` 指令時，若沒有特別指定 `PROFILE` 的名稱，則會使用 `default`
 這一組的資訊來存取，否則就會根據其名稱來尋找特定的數值

# Usage
有了上述的概念後，接下來來看一下要如何設定 `aws cli`。
1. 執行時帶入參數 `--profile` 來指定要使用的 `PROFILE`
2. 透過環境變數 `AWS_PROFILE` 來指定要使用的 `PROFILE`

# Solution

以(1)為範例的話，我們可以透過所有的指令都要補上 `--profile` 來使用，但是這種方式使用上會變成你所有指令都要確保有加上該參數，實務上我個人是沒有很喜歡，覺得管理上比較麻煩。
譬如
```bash=
aws iam list-roles --profile Account1
aws iam list-roles --profile Account2
aws iam list-roles --profile Account3
```

如果今天有一些客製化的腳本，變成腳本中所有的指令都要注意有使用到這個參數，對於一個多人維護的腳本來說，我覺得被忽略的可能性非常高

以(2)為範例的話，就是每次使用前要確保當前的環境變數是什麼，這種情況下每個指令的使用就可以盡量簡單，讓呼叫者自己去確保該指令的面對對象
```bash=
export AWS_PROFILE=account1
aws iam list-roles
AWS_PROFILE=account2 aws iam list-roles
```
套入環境變數的方法滿多種的，上述的方式都可以將該環境變數傳遞到 `aws` 的指令列中

但是使用這個方法的困難處就在於預設的情況下，你有時候會不知道自己到底當前是什麼樣的 `Profile`, 這時候就可以搭配 `shell prompt` 去客製化顯示當前的 `AWS_PROFILE` 是什麼。

為了解決這個問題，其實可以修改自己習慣使用的 `shell` 設定檔，搭配一些自行撰寫的 `shell function` 來動態調整當前的 `AWS_PROFILE` 環境變數，並且將當前的數值顯示在 `shell` 的提示命令列上。

就在我完成該功能沒多久後，就發現其實 `oh-myzsh` 內建的 `plugin` 有一個幾乎完全一樣的實現功能，這意味其實我根本不需要自己寫，直接啟動該 `plugin` 即可

# oh-my-zsh aws
## Usage
打開該 aws 套件後，使用上大概會是如下
```bash=
<aws:Account1>╭─hwchiu@hwchius-MBP ~
╰─$
```
使用指令 `asp` 搭配 `tab` 可以列出目前所有設定過的 `aws profile`
```bash=
<aws:LFCORD>╭─hwchiu@hwchius-MBP ~
╰─$ asp
Account1 Account2 Account3 Account4
```

同時該 `aws plugin` 也會幫忙設定安裝 `aws aws_zsh_completer`, 意味你可以透過 `tab` 的方式來更方便的使用 `aws cli`

## Configuration
基本上參考 `on-my-zsh` 的設定方式，先到 `$HONE/.zshrc` 中將 `plugin` 的選項加入 `aws` 即可，譬如
```bash=
...
# Which plugins would you like to load?
# Standard plugins can be found in ~/.oh-my-zsh/plugins/*
# Custom plugins may be added to ~/.oh-my-zsh/custom/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(git aws)

source $ZSH/oh-my-zsh.sh
...
```

這邊有一個要注意的是，預設的 `aws plugin` 使用的是 `RPROMPT` 而非 `PROMPT`, 這意味他的顯示會是在畫面最右邊，而非左邊。
此外也要注意妳選用的 `theme` 會不會幫你把 `PROMPT` 給蓋掉，導致功能失效。
譬如我選擇的主題是 `bira`，所以 `PROMPT` 以及 `RPROMPT` 都會被覆蓋掉導致 `aws plugin` 不會成功。

因此我最後還是自己修改 `$HOME/.zsrhc` 來處理
```bash=
...
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"
PROMPT='$(aws_prompt_info)'"$PROMPT"
```
詳細的更多實作可以直接參閱 [$HOME/.on-my-zsh/plugin/aws](https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins/aws) 裡面的介紹

藉由這個套件的幫助，我平常就會使用 `asp` 來切換不同的 `AWS Account`, 同時透過 `shell PROMPT` 來知道當前使用的 `PROFILE`，避免在不同的 `Account` 執行錯誤的動作

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