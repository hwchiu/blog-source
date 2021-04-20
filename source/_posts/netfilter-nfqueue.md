---
title: NFQUEUE drop UDP packets
keywords: 'Linux,Kernel,Netfilter,nfqueue'
date: '2016-03-20 15:32'
comments: true
tags:
  - Network
  - Netfilter
  - NFQUEUE
  - Linux
  - Kernel
abbrlink: 56894
description:
---

## Introduction
此篇文章用來記錄最近遇到的一個問題，在一個執行 NFQUEUE 的系統上，當一個尚未被 conntrack 紀錄的連續 UDP 封包經過系統且都經過 NFQUEUE 處理後，第二個 UDP 封包都會遺失的問題。

## NFQUEUE
- 一種 Queue，由 netfilter (ipables) 所提供的一種 target，能夠將封包內容藉由 netlink/nmap 送到 user-space 去，大部分的 IPS/IDS 都會藉由此方式來分析封包，如 suricata。
- User space 有對應的 library 可以用來接收此封包，參考此[***link***](http://www.netfilter.org/projects/libnetfilter_queue/)
- 相關的 tutorial 可[***參考***](https://home.regit.org/netfilter-en/using-nfqueue-and-libnetfilter_queue/)

    <!--more-->

## 系統資訊
- Linux kernel 3.6

## Problem
- 系統在 filter chain 的 forward table 中加入一條 iptables 將封包導向 NFQUEUE。
- User space 可參考此[project](https://github.com/irontec/netfilter-nfqueue-samples)，使用一個最簡易的 sample，將封包收到後就送回 kernel
- 系統運行 NAT
- 當有連續 UDP 封包經由系統往外送出時，可觀察到第二個 UDP (可能更多)都會遺失
	- 此連續 UDP 封包必須還沒有被 kernel 的 conntrack 給紀錄
- 若系統沒有運行 **NFQUEU** ,則此問題不存在

## 問題觀察
## #若沒有運行 NFQUEUE，為什麼封包正常
- 請參考此流程圖
![](https://lh3.googleusercontent.com/-zE8ZORw-gM4/Vu7F8Cu-WSI/AAAAAAAAFKk/zVj-3sFOErg8Jrnn0XHOewe5-m5QjmGggCCo/s806-Ic42/without_nfqueue.png)
- 第一個封包進入到系統時，於**1**處時，會判斷該封包是第一次建立 connection，所以是 unconfirm 的狀態，於**3**的狀態時，會去將該 connection 給 confirm，並且將該資訊給存入 kernel 的 hash 之中。

- 第二個封包進入時，於**1**處時去判斷，就會知道該 connection 已經建立了，所以就不會進入到**3**，後續封包都按照此流程傳送。


## #若運行 NFQUEUE，為什麼封包會丟棄
- 請參考此流程圖
![](https://lh3.googleusercontent.com/-timTg7c3jfU/Vu7GBU6s6vI/AAAAAAAAFKo/amgicDdly0EjTI8faCi8D3jg1HSk2vNWwCCo/s1106-Ic42/with_nfqueu.png)
- 第一個封包進入到系統時，於**1**處時，會判斷該封包是第一次建立 connection，所以是 unconfirm 的狀態，接下來就透過 netlink 要送到 user space 去。

- 第二個封包進入到系統時，由於第一個封包還沒有被 kernel 內的 nf_conntrack_confirm 處理完畢(可能封包還在 user space)，所以於1處時，也會判斷封包是第一次建立 connection，是 unconfirm 的狀態。

- 當 user space 將封包打回 kernel 後，會於**3/4**開始處理，會從先前的 queue 將 skb 所記錄 conntrack 的資訊給取出，所以這兩個封包都會認為自己是 unconfirm 的狀態

- 第一個封包接下來會走完全部的路途，並且送出去

- 第二個封包當走到 nf_conntrack_confirm 時，會因為覺得自己是 unconfirm 的，所以呼叫 __nf_conntrack_confirm 去處理。
	code: [ref](https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/tree/include/net/netfilter/nf_conntrack_core.h#n69)

- 當第二個封包跑到 __nf_conntrack_confirm 時，會嘗試將自己的 conntrack 給加入到 kernel hash中，但是第一個封包已經加入過了，所以 kernel 會覺得你有病，就將該封包給丟棄了。

## #結論
- 此問題發生的根本在於 conntrack 的衝突，當 conntrack 的結果已經被 kernel 紀錄的情況下，有第二個封包嘗試將 conntrack 再次存到 kernel 中，此封包就會被丟棄
	code:  [ref](https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/tree/net/netfilter/nf_conntrack_core.c?h=linux-3.6.y#n511)
- 看過 linux 4.4 的程式碼，在 netlink 接收也沒有針對收到的封包去重新處理 conntrack 的問題，我想是 UDP 掉封包是合理的，而 TCP 會自己重傳，所以上層的應用程式不會有感覺。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
