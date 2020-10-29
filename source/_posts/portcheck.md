---
layout: post
title: 檢查port使用情況
date: '2013-03-29 12:33'
comments: true
tags:
  - System
  - Network
  - FreeBSD
  - Windows
  - Linux
abbrlink: 33969
---

有時候根據應用需求，會需要針對去檢查目前系統上有哪些port正在被使用

#**[FreeBSD]**

可以使用 sockstat 這個command 來檢查系統上port的使用。

>USER COMMAND PID   FD PROTO LOCAL ADDRESS FOREIGN ADDRESS

>root     cron 93468     4   udp4           *:638                          *:*

在預設的情況下，會輸出

使用者名稱，執行的程序，該程序的pid，在該程序中使用該port的file descriptor是多少 使用何種協定，以及address

如果使用 sockstat -4lP tcp 就可以找出 使用tcp & ipv4 ，並且正在listen的port

這對於要尋找是否有人在寫**Socket programming**來說是很方便的。

詳細的可以man sockstat
***

<!--more-->

#**[Linux]**
可以使用 netstat 這個工具來檢視，搭配一些參數還可以看到該 port 被那些 process 使用
```
netstat -anptn
tcp        1      0 127.0.0.1:40147         127.0.0.1:36524         CLOSE_WAIT  7147/vim
tcp        1      0 127.0.0.1:58289         127.0.0.1:52849         CLOSE_WAIT  19421/vi
...
```

#**[Windows]**

可以使用netstat來檢視，netstat能夠顯示的資訊非常的多，為了精簡我們的需求，必須去過濾這些資訊

在windows上使用find這個指令，類似於UNIX中grep的功能

舉例來說，netstat -an |find /i “listening" 這個指令

netstat  -an 會顯示所有連線以及正在監聽的port，並且以數字的形式來顯示IP以及PORT

find /i “listening" 則會以不區分的方式去搜尋每一行，若包含listening則將該行印出

EX:

>TCP 192.168.1.116:139 0.0.0.0:0 LISTENING

>TCP 192.168.1.116:49156 216.52.233.65:12975 ESTABLISHED


ref:
[www.microsoft.com/resources/](http://www.microsoft.com/resources/documentation/windows/xp/all/proddocs/en-us/find.mspx?mfr=true)

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/JPwq4znr1

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=7EP1KY3

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