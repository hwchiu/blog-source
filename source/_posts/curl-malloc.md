---
title: curl with fewer malloc
tags:
  - System
abbrlink: 20597
date: 2017-05-05 18:22:03
---
不久前有一篇文章[https://daniel.haxx.se/blog/2017/04/22/fewer-mallocs-in-curl/](https://daniel.haxx.se/blog/2017/04/22/fewer-mallocs-in-curl/)指出， curl 開發者嘗試將 malloc 呼叫的次數減少，結果對整體的影響帶來的顯著的提升

使用 `curl http://localhost/512M` 當作第一個比較
原始版本的 curl 關於 malloc 相關數據如下
```
Mallocs: 33901
Reallocs: 5
Callocs: 24
Strdups: 31
Wcsdups: 0
Frees: 33956
Allocations: 33961
Maximum allocated: 160385
```
而修改後的版本為
```
Mallocs: 69
Reallocs: 5
Callocs: 24
Strdups: 31
Wcsdups: 0
Frees: 124
Allocations: 129
Maximum allocated: 153247
```

<!--more-->

比較起來可以發現， malloc 呼叫的次數有急遽的下降，從 33901 降落到 69，而整體使用的記憶體也少了 7KB 左右
此外，若比較兩者傳輸的速度，抓取一個 80GB 的檔案
```
Original: 2200MB/sec
Modified: 2900MB/sec
```
在傳輸方面提升了 30% 左右的速率，非常驚人
若使用 time 指令來比較新舊版本抓取 80G 檔案的差別
```
Old code:

real    0m36.705s
user    0m20.176s
sys     0m16.072s
New code:

real    0m29.032s
user    0m12.196s
sys     0m12.820s
```

修改相關的 commit 如下
- [llist: no longer uses malloc](https://github.com/curl/curl/commit/cbae73e1dd95946597ea74ccb580c30f78e3fa73)
- [multi: make curl_multi_wait avoid malloc in the typical case](https://github.com/curl/curl/commit/5f1163517e1597339d)

簡單來說就是將 malloc 的部分都拔除，盡量使用 stack 來提供記憶體，藉此減少呼叫 malloc 的部分。
主要是因為 curl 在傳輸的過程中，會有非常大量且小空間的 malloc 被呼叫到，這部分拖慢的整體的運行速度
