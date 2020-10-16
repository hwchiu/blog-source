---
layout: post
title: Git 筆記
date: '2014-07-28 13:04'
comments: true
tags:
  - Git
  - System
abbrlink: 20539
---
Basic
-----
- commit所使用的編輯器會依照下列優先度去選擇，
	1. GIT_EDITOR 環境變數
  2. core.editor 的設定
  3. VISUAL 環境變數
  4. EDITOR 環境變數
  5. vi 指令

<!--more-->
- 變動檔案請用 `git mv`，使用`git rm`要注意檔案系統內的檔案會被真的刪除。
- `git log`可以列出簡略的coommit資訊
- `git show [commit id]` 可以看詳細的commit資訊，可以加上commit ＩＤ來指定特定的commit
- `git show-branch --more=10` 可以看當前bracnh的詳細commit資訊，由**--more**控制數量

Configuraion
------------
總共有三種設定方式，優先度如順序
- .git/config， 可以用 `--file`或是預設的方式操作
- ~/.gitconfig， 可以用 `--global`操作
- /etc/gitconfig，可以用 `--system`操作
```sh
git config --global user.name "hwchiu" (2)
git config user.email "hwchiu@cs.nctu.edu.tw" (1)
```
- 可以透過 `git config -l`列出當前所有的設定
- 可以透過 `--unset`來移除設定
```sh
git config --unset --global user.name
```


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