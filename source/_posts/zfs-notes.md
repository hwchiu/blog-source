---
title: ZFS 筆記
keywords: 'FreeBSD, ZFS'
date: '2013-10-12 17:53'
comments: true
tags:
  - FreeBSD
  - System
  - ZFS
abbrlink: 61488
description:
---
之前機器因為ZFS空間滿了，因為平常有再作snapshot的緣故，導致東西都刪除不了
因為刪除的時候都會有一些metadata的寫入，導致整個zfs動彈不得，這時候就花了很多時間再研就怎麼處理
這邊稍微記錄一下ZFS相關得操作。
ZPOOL的來源可以是device也可以是files,這邊就用兩個檔案當作來源。

<!--more-->

## Files
- `sudo dd if=/dev/zero of=/zfs1 bs=1M count=256`
- `sudo dd if=/dev/zero of=/zfs2 bs=1M count=256`

## Zpool
- create a mirror pool
	- `zpool create ftphome mirror /zfs1 /zfs2`
- destroy a pool
	- `zpool destroy ftphome`
- check zpool status
	- `zpool status <pool>`
- export pool ( 把某些pool export出去，暫時不使用)
  - `zpool export ftphome`
- import pool ( 把被export 的pool 重新import回來)
	- `zpool import -d /  ftphome`  (用-d指定你檔案的位置，預設會去吃/dev/)
  - 以我的範例來說，當import回來後，名稱會變成 `//zfs1`, `//zfs2`，多了一個/，原因不明中。
- attach ( 只能對mirror使用)
	- `zpool attach ftphome /xxx`
- detach ( 只能對mirror使用)
  - `zpool detach ftphome /zfs1`

還有`offline`,`online`,`remove`...，剩下的就要用的時候去man zpool,還滿詳細說明的。


## ZFS database
- set attributes `zfs set key=value <filesystem|volume|snapshot> `
  - `zfs get compression ftphome`
  - `zfs set mountpoint=/home/ftp ftphome`
- get attributes `zfs get key <filesystem|volume|snapshot> `
  - `zfs get compression ftphome`
- snapshot
	- `zfs snapshot ftphome@today `
  - `zfs list -t snapshot`

## 其他
- 假如你的ZFS有使用snapshot同時空間又滿的話，這時候會發現所有檔案都會刪除失敗，都會得到空間不足的訊息,這邊稍微模擬一下該情況，並且想辦法解決此問題。

### 模擬情況

**snatshot 該zfs**
- `zfs snapshot ftphome@today`
- `zfs list -t snapshot`   看一下是否有成功

**塞爆該空間**
- `zfs list` 看一下還剩下多少空間
- `dd if=/dev/random of=/home/ftp/file bs=1M count=256`
- `cd /home/ftp`
- `rm file`  => 應該會得到 ` No space left on device `空間不足的訊息。

### 解決問題
ZFS 變大容易(多塞個硬碟即可)，變小困難(幾乎無法)，因此當ZFS的硬碟滿的時候，有兩種做法
1. 再加入兩個新的硬碟，然後合併到目前的zpool,可是這樣就會變成有兩份mirror。
2. 準備兩個更大的硬碟，把原本的zpool內的data全都複製過去。
這邊使用第二種做法

**先幫本來的pool加入一個檔案，增加本來的空間，如此一來才可以做更多操作**
- `dd if=/dev/zero of=/zfs5 bs=1M count=128`
- `dd if=/dev/zero of=/zfs6 bs=1M count=128`
- `zpool add ftphome mirror /zfs5 /zfs6`
- `zfs list`
   (此時可以看到本來的空間變大了)

**創造一個更大的zpool來取代**

- `dd if=/dev/zero of=/zfs3 bs=1M count=512`
- `dd if=/dev/zero of=/zfs4 bs=1M count=512`
- `zpool create ftphome3 mirror /zfs3 /zfs4`
- `zfs set compression=gzip-9 ftphome2`

**把資料複製過去**
- `zfs snapshot ftphome@send`
- `zfs send ftphome@send | zfs receive -F ftphome2`
- `zfs list` 看一下大小是否相同

**mount新的，舊的砍掉**
- `zfs umount ftphome`
- `zfs set mountpoint=/home/ftp/ ftphome2`
- `zpool destroy ftphome`

做到這邊，就算完成了，成功的把本來的資料複製過去。
如果想要改變zpool的名稱，可以用`export`跟`import`來改名稱。

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