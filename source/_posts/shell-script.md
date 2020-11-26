---
title: Shell Script 筆記
keywords: 'Linux,Shell Script, pipe'
date: '2013-11-24 12:39'
comments: true
tags:
  - System
  - ShellScript
abbrlink: 17288
description: 記錄一些之前寫 Shell Script 作業的心得與筆記

---

# Preface
本篇文章是用來記錄以前修課關於 Shell Script 的作業

# Introduction
用Unix的指令，透過pipe的方式完成下列要求
- 計算當前目錄底下資料夾的總數
- 計算當前目錄底下檔案總數
	* 只有計算regular file.不考慮FIFO、LINK
- 計算所有檔案大小和 (Byte)
- 顯示前五大的檔案名稱
- 只能使用PIPE，不能使用 `$$` `;` `||` `&` `>` `>>` `<`

# Implement
- 使用`ls`來取得所有資料夾跟檔案的資訊
	* -l  可以顯示詳細資訊，這邊我們要取得的是 檔案大小
  * -R  遞迴的往每個資料夾繼續往下找
  * -A  不要把`.`跟`..`給顯示出來，因為這種當前目錄的東西我們不需要
- 使用`sort`來幫忙排序，找出檔案大小前五個
	* -r  排序結果反過來，由大到小排序
  * -n  排序的時候，採用數字的方式去排序，不使用字母大小去排序
  * -k  指定第幾個欄位要排序
- 使用`awk`作最後的處理，找出前五大，印出所有檔案大小和
  * 因為再`ls -l`的結果中，會有很多資訊，包含 `./cs/.svn/pristine/74:` 或者 `total 28`，所以awk再處理的時候，先用NF判斷該行的欄位數，至少要有9個欄位才處理 `if(NF>=9)`
  * 接下來針對檔案是資料夾還是檔案，做全部的計數，可以由 `-rw-r--r--` 的第一個欄位來決定，如果是d就代表資料夾，否則就是檔案。 這邊我使用 regular expression來判斷 `($1 ~/^d/)? (dir=dir+1) : (file=file+1)(size=size+$5)`，此外如果是檔案得話，就順便把大小也計算一下
  * 執行過程中，因為剛剛已經排序過了，所以 前六行都把大小印出來， `if(NR<6) print NR": "$5" "$9}`
  * 最後就把所有資訊都列印出來

``` sh

ls -RlA | sort -rnk 5 | awk '{ if(NF>=9) ($1 ~/^d/)? (dir=dir+1) : (file=file+1)(size=size+$5); if(NR<6) print NR": "$5" "$9} END{ print "Dir = "dir"\n" "    File = " file"\n" "total = "size}'

```

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4N7G

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