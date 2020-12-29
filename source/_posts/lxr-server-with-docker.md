---
title: docker image for lxr server
keywords: 'LXR, Ubuntu, Docker, SourceCodeIndex'
tags:
  - Docker
  - Lxr
  - Ubuntu
  - SourceCode
  - System
abbrlink: 16582
date: 2017-06-07 01:54:30
---


之前曾經發過一篇文章[LXR Server With Multiple Projects](https://www.hwchiu.com/lxr-setup-with-multiple-projects.html)，主要介紹如何在 **Ubuntu** 上面安裝 **lxr** 並且支援多個 **projects**。由於整個 **lxr** 的安裝過程複雜，除了本體外還牽扯到不少第三方程式套件，如 **perl**, **database**, **www server**，且大部份都是安裝完畢後就再也不會更動，唯一的更動應該就是更換要被 indexing 的 project而已。

上述這種使用情境我覺得非常適合使用 **docker** 來建置一個 image， image 將所有相關的套件都全部安裝完畢，並且套用一個預設的設定檔當作基礎設定。

接下來每次運行的時候，只要將 **source code** 的位置以及相關的版本資訊都帶入到 `docker run` 的參數中，就可以跑起一個 **www server** 並且提供該 project 的網頁服務。

<!--more-->
我將弄好的 docker image 放到 [docker-lxr](https://hub.docker.com/r/hwchiu/docker-lxr/])這邊，由於要支援多個 projects 上面，在 **lxr.conf** 中沒那麼好設定，同時又希望保留彈性，能夠在 **docker run** 的時候去處理，所以這邊先考慮 **single project** 的情況。
使用上只有幾個部分需要注意
- source code 的位置，裡面每個版本都要有一個屬於自己的資料夾，且資料夾名稱就是版本號
- ip:port 對外使用的 ip:port 資訊。

假設我將 **ceph code** 分成三個版本，都放在 **/tmp**底下
```bash
>ll /tmp
total 12K
drwxr-xr-x 23 root root 4.0K Jun  6 13:40 master
drwxr-xr-x 23 root root 4.0K Jun  6 13:52 v12.0.3
drwxr-xr-x 23 root root 4.0K Jun  6 13:48 v9.2.1
```
且假設對外的 **ip address** 是 10.2.3.4

則使用方式很簡單
```
docker pull hwchiu/docker-lxr
docker run --name lxr -it -v tmp:/source   -p 800180  hwchiu/lxr:single 10.2.3.4 master v12.0.3 v9.2.1
```

更多的詳情可以參考 [docker-lxr](https://hub.docker.com/r/hwchiu/docker-lxr/]) 或是 [github/docker-lxr](https://github.com/hwchiu/docker-lxr)

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
