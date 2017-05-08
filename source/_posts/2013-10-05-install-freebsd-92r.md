---
layout: post
title: 'Extend freebsd-ufs system'
date: 2013-10-05 09:07
comments: true
categories: [System, freebsd]
tags:
	- System
	- FreeBSD
---
假設你今天在VM上安裝FreeBSD，然後因為硬碟空間不夠，變透過VM的設定去擴充硬碟空間
那使用 `gpart show`你會得到下列資訊 

          34   62914493  ada0   GPT  (232G)
          34        128  1      freebsd-boot
         162   39845760  2      freebsd-ufs (19G)
    39845922    2097152  3      freebsd-swap (1G)
    41943074   20971453         - free - (10G)
    
    
<!--more-->

會發現新增加的10G並沒有直接增加到原本的系統中，而是一個free的狀態，需要手動去合併。
那這時候我們就要把原本的ufs跟新增的區塊給合併。

但是由於中間卡了一個`swap`的區塊，所以我們要先把該swap給砍掉，
然後重新建立一個swap的區域，接者再把兩個ufs的部分合併。

###Step###
**由於我們要對root partition去進行操作，所以請先進入live cd的環境**

- 先刪除本來的swap空間 `gpart delete -i 3 ada0`
- 擴大本來的ufs `gpart resize -i 2 -s 20G ada0`
- 用剩下的空間再創立一個swap `gpart add -t freebsd-swap ada0`    

              34   62914493  ada0   GPT  (232G)
              34        128  1      freebsd-boot
             162   60817408  2      freebsd-ufs (29G)
        60817570    2096957  3      freebsd-swap (1G)
- 使用 `growfs /dev/ada0p2` 來把空間真正的擴大

