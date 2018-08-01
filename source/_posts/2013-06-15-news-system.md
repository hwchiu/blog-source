---
layout: post
title: news-system
date: '2013-06-15 09:25'
comments: true
tags:
  - System
  - FreeBSD
abbrlink: 27654
---

用來紀錄2013/06/12日 news系統發生的問題

<!--more-->



當天經學長告知才發現news前天開始就一直無法把信轉出去的情況，然後可以看到

news.daily被queue住非常多，完全沒有半個可以順利結束，


此時在~news/spool/incoming/目錄下可以看到非常多卡住的郵件

會發生這個原因是因為innd死了，但是nnrpd沒有死，所以所有的信件都會被存放在該位置

系統不知道出什麼鬼問題，innd完全無法連線，透過telnet localhost 443去嘗試都沒有辦法建立連線

innd完全死掉，重啟後服務依然跑不起來，而且innd還要透過Kill才能將該process給砍掉，最後跟學長討論的決定

就把整個機器重開了，重開後一切正常，此時使用rnews -U 可以把~news/spool/incoming中的信件全部給送出去

此時就會造成板上一口氣噴出很多信件而且時間點都不對XDD
