---
title: ZNC IRC bouncer
keywords: 'ZNC, IRC'
date: '2014-07-23 02:22'
comments: true
tags:
  - FreeBSD
  - IRC
abbrlink: 58768
description: ZNC IRC Bouncer 安裝筆記

---


# Environment
- FreeBSD 10.0-Release

# INSTALL
## pkgng
1. pkg install znc

## Ports
1. cd /usr/ports/irc/znc
2. make config
3. make install & clean

# Config
znc --makeconf
- add listen port.
- add user
- add network. ex: freenode
	-	add irc server. ex: irc.freenode.net
	- you can add the default channel passowd  by a key=? option in znc.conf

# Usage
## Find a irc client
- [AndChat on Android](https://play.google.com/store/apps/details?id=net.andchat)
- [kiwiirc on Web](https://kiwiirc.com/)

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