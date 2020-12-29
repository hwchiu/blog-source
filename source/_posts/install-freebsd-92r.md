---
title: Extend freebsd-ufs system
keywords: 'FreeBSD, extend, filesystem, ufs, capacity'
date: '2013-10-05 09:07'
comments: true
tags:
  - System
  - FreeBSD
abbrlink: 58332
description: 本文要介紹如何在 FreeBSD 的環境下，如果遇到空間不夠，然後透過 VM Manager 的方式去擴充一顆硬碟空間時，要如何將該硬碟空間跟本來的硬碟空間給合併成一個更大的儲存空間來使用。這個情境我個人還滿長遇到的，因為有時候透過 VM 去創立系統時，一開始沒有想到可能會使用到的空間大小，結果使用後硬碟馬上就空間不足了。雖然可以透過 VM 的管理方式擴充舊有的硬碟空間大小。本文針對這部分筆記一下使用的指令以及概念。

---

# Preface
假設你今天在VM上安裝FreeBSD，然後因為硬碟空間不夠，變透過VM的設定去擴充硬碟空間
那使用 `gpart show`你會得到下列資訊
```
          34   62914493  ada0   GPT  (232G)
          34        128  1      freebsd-boot
         162   39845760  2      freebsd-ufs (19G)
    39845922    2097152  3      freebsd-swap (1G)
    41943074   20971453         - free - (10G)
```

會發現新增加的10G並沒有直接增加到原本的系統中，而是一個free的狀態，需要手動去合併。
那這時候我們就要把原本的ufs跟新增的區塊給合併。

但是由於中間卡了一個`swap`的區塊，所以我們要先把該swap給砍掉，
然後重新建立一個swap的區域，接者再把兩個ufs的部分合併。

# Note
**由於我們要對root partition去進行操作，所以請先進入live cd的環境**

- 先刪除本來的swap空間 `gpart delete -i 3 ada0`
- 擴大本來的ufs `gpart resize -i 2 -s 20G ada0`
- 用剩下的空間再創立一個swap `gpart add -t freebsd-swap ada0`
```
              34   62914493  ada0   GPT  (232G)
              34        128  1      freebsd-boot
             162   60817408  2      freebsd-ufs (29G)
        60817570    2096957  3      freebsd-swap (1G)
```
- 使用 `growfs /dev/ada0p2` 來把空間真正的擴大
