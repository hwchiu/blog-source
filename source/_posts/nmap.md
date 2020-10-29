---
layout: post
title: nmap
date: '2013-06-15 09:25'
comments: true
tags:
  - System
abbrlink: 16805
---


nmap是一個linux下的工具

nmap - Network exploration tool and security / port scanner
<!--more-->


這邊記錄一下nmap的用法

nmap -sP 140.113.214.79/27

-sP: Ping Scan - go no further than determining if host is online
用ping去掃目標內的所有IP，並顯示有回應的IP，所以若對方是windows7且沒有打開ping的回應，則也會被當作host down

nmap -sL 140.113.214.79/27

-sL: List Scan - simply list targets to scan
只是單純的列出對方的hostname以及IP，不送出任何封包去檢測

nmap -O 140.113.214.94
nmap -A 140.113.214.94

-O: Enable OS detection
-A: Enables OS detection and Version detection, Script scanning and Traceroute

掃描對方主機的OS系統

nmap -PS/PA/PU/PY[portlist] 140.113.214.94

-PS/PA/PU/PY[portlist]: TCP SYN/ACK, UDP or SCTP discovery to given ports

用不同的方式去掃描特定的PORT。

- PS 用TCP 搭配 SYN FLAG去偵測。
- PA 用TCP 搭配 ACK FLAG去偵測。
- PU 用UDP去偵測。
- PY 用SCTP去偵測。


nmap -sS/sT/sU 140.113.214.94

採用不同的方式去掃描所有port。

- sS (TCP SYN scan) .
- sT (TCP connect scan)
- sU (UDP)

nmap -v 140.113.214.94
顯示出詳細一點的資訊


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