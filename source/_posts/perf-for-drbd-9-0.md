---
title: perf_for_drbd_9.0
tags:
  - DRBD
  - Perf
  - System
  - Linux
  - Kernel
  - Fio
keywords: 'DRBD,Linux,System,Kernel,Fio'
abbrlink: 51237
date: 2017-05-19 17:57:24
---

本文主要嘗試分析 drbd(9.0) 於 **kernel**運行時的效能分析，希望藉由 **perf** 這個 tool 來分析整個程式運行的狀況，藉此觀察其運行時各 function 的比例。

### Testing Environment
為了進行效能上的分析，首要條件就是先將 **DRBD** 給衝到效能瓶頸才有機會去觀察，所以本文採用下列的環境與工具來進行這項作業

<!--more-->

#### Environment
CPU: Intel(R) Xeon(R) CPU E5-2695 v3 @ 2.30GHz
Storage: Non-Volatile memory controller(NVME)
Tool: [fio](https://github.com/axboe/fio)
OS: Ubuntu 16.04 with linux 4.4.0-78-generic

#### Setup
為了更方便觀察 drbd 的運行，我們將 drbd 創造的 kernel thread 都分別綁在不同的 cpu 上，這樣可以讓每隻 kernel thread 盡可能去使用cpu。

1. 透過 `ps` or `htop` 取得 kernel thread 的 **pid**,這邊可以關注的有
    - drbd_s_r0 (sender)
    - drbd_r_r0 (receiver)
    - drbd_as_r0 (ack sender)
    - drbd_a_r0 (ack receiver)
    - drbd_w_r0 (worker)
2. 透過 `taskset` 這個指令將上述程式分別綁到不同的 cpu 上
```
taskset -p 0x100 18888
...
```

#### Stress
本文使用 **fio** 來進行資料的讀取，下方提供一個簡易的 fio 設定檔，可依照自行的環境變換修改。

``` config
[global]
iodepth=512
numjobs=3
direct=1

time_based
runtime=30
group_reporting
size=5G
ioengine=libaio

filename=/mnt/beegfs/fio1.test
[rrw]
bs=4k
rw=randrw
rwmixread=75

[rr]
bs=4k
rw=randread

[rw]
bs=4k
rw=randwrite

[sr]
bs=64k
rw=read

[sw]
bs=64k
rw=write
```

我們 fio 採用 client/server 的架構，要是可支援多台 client 同時一起進行資料讀取，提供更高的壓力測試。

假設該設定檔名稱為 **fio.cfg**，並且放置於 **/tmp/fio.cfg**
則首先在 **node-1** 上面執行下列指令以再背景跑一個 fio server
```
fio -S &
```
接下來要運行的時候，執行下列指令來運行 fio，其中若想要改變測試的類型，可透過 **--secion**進行切換。

```
/fio --client=node-1 /tmp/fio.cfg --section=rw

```
這時候可以透過 **htop** 以及 **iostat** 的資訊去觀察，如下圖
當前透過 **iostat** 觀察到的確對 **drbd0** 有大量的讀寫動作
![](http://i.imgur.com/C7EKH2f.jpg)
同時由 **htop** (記得開啟 kernel thread觀察功能)，可以看到 **drbd_s_r0** 以及 **drbd_a_r0** 都各自吃掉一個 cpu，大概都快接近 100% 的使用率。
![](http://i.imgur.com/neMXdHE.jpg)

#### Profile
有了上述的環境後，我們就可以準備來分析 drbd 程式碼運行狀況。

###### Environemnt
這邊使用 **perf** 這套程式來分析，基本上 **kernel** 新一點的版本都已經內建此功能了，比較舊的 **kernel** 則需要自己重新開啟該 **kernel config**然後重新 build kernel，所以這邊使用 **Ubuntu 16.04 with linux 4.4.0-78-generic** 相對起來非常簡單。

直接執行 `perf` 這個指令，若系統上有缺少 **linux-tools-4.4.0-78** 相關 tool 的話會有文字提示你，如下所示，按照提示使用 **apt-get** 將相關的套件安裝完畢後，就可以使用 **perf** 了。
```
WARNING: perf not found for kernel 4.4.0.78

  You may need to install the following packages for this specific kernel:
    linux-tools-4.4.0-78-4.4.0-78
    linux-cloud-tools-4.4.0-78-4.4.0-78
```

##### Run
perf 的功能非常強大，可以參考 [wiki](https://perf.wiki.kernel.org/index.php/Tutorial), 這邊我們使用 **perf top** 的方式來觀察結果。
為了可以順便觀看 **call graph** 的過程，再執行`perf`的時候要多下`-g`這個參數

指令為 **perf top -g -p $PID**，如 **perf top -g -p 18888**。

在這邊我嘗試觀察 **drbd_a** 這隻，結果如下列

##### drbd_a
這邊可以觀察到三隻吃比較兇的 **function** 都吃很兇,分別是 **native_queue_spin_lock_slowpath** 、 **tr_release** 以及 **idr_get_next**。

這邊比較麻煩的是你看這個只能知道就是卡在 spin_locK，系統上是不是 multithread，然後有太多的資料要搬移導致 spin_lock ? 這些搬移的資料是誰放進去的?，這些資料是什麼?

以及更多的問題都必須要看程式碼去理解其整體設計架構，才有辦法講出一套完整的流程說明這個結果。

這部份等之後能夠完整理解整個 drbd 的 write-path 或是 read-path 時再重新來看一次這張圖，到時候應該會有完全不一樣的思維。

![](http://i.imgur.com/Bi1ZKqn.jpg)

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