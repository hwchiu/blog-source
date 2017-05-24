---
title: Docker image for Hexo (一)
date: 2017-05-24 09:46:35
tags:
	- Docker
	- Container
	- Hexo
	- System
keywords: Docker,Hexo,System,Container
---

本 blog 目前是採用 **hexo** 作為主要開發，然而 **hexo** 我覺得最大的缺點就是屬於靜態網頁類型，所以只要每次換一個開發裝置，譬如桌電,筆電等，就必須要將整個環境重新建置。
整個 **hexo** 是由 **nodejs** 所組成的，所以其實在安裝上也不會有什麼太大的麻煩，但就是覺得相同的事情做過很多次，其實還滿蠢的。
所以決定採用 **docker** 的方式，將該開發環境做成一個 **docker image**，之後有需要直接抓取該 **image** 下來使用即可。

因此本文接下來將著重在如何使用 **docker** 指令創立一個 **image**，並且將其上傳到 **docker hub**上。
<!--more-->

## Environment
- OS: Ubuntu 16.04
- Docker version 1.12.6, build 78d1802

## Install
首先，我們要創建一個 **Dockerfile**，這個檔案除了用來描述該 **Docker image**的基本資訊外，也包含了當該**Docker image**被創建時，要執行些什麼指令，可以用來更新套件中心，安裝套件甚至是啟動特定服務都可以。
我們先從最基本的資訊開始。
- 透過 **mkdir** 創建一個資料夾，我們接下來要在此放 **Dockerfile**
```bash
mkdir hexo_image
```
- 使用習慣的編輯器(vim/nano/ee) 開啟 **Dockerfile**
``` bash
cd hexo_image
vim Dockerfile
```
- 接下來開始編輯內容，在該檔案內, **#**開頭的代表註解。
    - 首先透過**FROM**來指示該 **image** 的環境，這邊採用的格式是 **dis:version**，舉例來說可以使用 **ubuntu:16.04**
    - 接下來透過 **MAINTAINER** 來說明該 **image** 是由誰維護的。
```
FROM ubuntu:16.04
MAINTAINER hwchiu(sppsorrg@gmail.com)
```
- 接下來我們可以執行一些指令，譬如將一些基本指令完畢，這樣每次創建該 **image** 時，就可以有一些基本的環境
```bash
RUN apt-get update
RUN apt-get install -y net-tools
RUN apt-get install -y git
RUN apt-get install -y npm
RUN apt-get install -y vim
```

## Build
上述事情都完畢後，接下來我們要透過 **docker build** 此指令去建立該 **image**，在創建之前，我們可以先透過 **docker images** 觀看系統上目前擁有的 **docker images** 資訊
```bash
sppsorrg@ubuntu:~/hexo_image$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```
這邊總共有四行欄位,基本上其含義就如同其名稱一樣，我們在創建自己的 **image** 的時候，可以指定該 **REPOSITORY** 以及 **TAG**。
接下來就來使用 **docker build** 來使用剛剛的 **Dockerfile**
``` bash
docker build -t myhexo:latest .
```
這邊使用到了兩個參數，分別是 **-t myhexo:latest** 以及 **.**
由指令說明可以知道 **-t**可以用來指示該 image 的名稱，分別對應到上述的 **REPOSITORY** 以及 **TAG**，而 **.** 則是說明想要使用的 **Dockerfile** 的位置在哪裡
```bash
-t, --tag value               Name and optionally a tag in the 'name:tag' format (default [])
```
接下來就等待一段時間，當跑完後，就可以再度使用 **docker images** 觀看創建好的 **images** 資訊了。

```bash
sppsorrg@ubuntu:~/hexo_image$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myhexo              latest              ef7f745a9126        43 minutes ago      481.7 MB
```

若想要刪除 **images**，可以透過 **docker rmi $imput** 來刪除，其中 **imput** 可以是 **name:tag** 或是 **IMAGE ID** 的形式。

## Run
一切都創建完畢後，接下來就要透過 **docker run** 這個指令來執行剛剛創立的 **docker image**。
這邊我們會使用三個參數，分別是 **--name**,**-i** 以及 **-t**
**-name**代表的是該創建出來的 **container**的名稱，之後 **rm/stop**等指令都可以使用此名稱操作，不然就要使用 **containerID**。
**-it**則是將該 **container**的 **stdin/stdout**都導出到外面的 **tty** 上面，因為我這個 **container** 之後是想要可以進去進行 **hexo** 的編輯，所以這邊就將其導出來。

```bash
docker run --name hexo -it myhexo
```

如果想要觀察這些 **container** 的狀況，可以使用 **docker ps -a**的方式觀察。

## Push
最後我們要將當前創好的 **image** 給上傳到 docker hub 上，請自行
往[docker hub](https://hub.docker.com/)創建帳號並且設定好一個 **repository**。
這邊要注意的是，我們這邊是上傳剛剛創建好的 **images** 而並非後續運行的 **container**，所以若有任何步驟是在 **container** 內執行的，則上傳上去的 **images** 並不會有任何修改。

首先，先透過 **docker login** 登入到遠方的 **docker hub**。
由於遠方 docker hub 上還沒有任何 image, 所以我們這邊要主動上傳剛剛創造的 **images**
接下來使用 **docker tag** 將剛剛創建的 **images** 指定到 **docket hub**上的帳號
```bash
sudo docker tag myhexo hwchiu/hexo
```
接下來就可以透過 **docker images** 觀察到系統上多了一個 **images**，其名稱是 **hwchiu/hexo**, 但是 **image ID**則會跟剛剛創立的一樣。

最後透過 **docker push myhexo** 則可以將該 **images** 給傳上 **docker hub**上的 repository 了。

所有步驟到這邊，已經可以創好一個簡單的 **docker image**，接下來就可以在 **Dockerfile** 內加入更多的指令，讓該 **image** 一創建時就有更完善的環境。
