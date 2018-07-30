---
title: Docker image for Hexo (二)
tags:
  - Docker
  - Container
  - Hexo
  - System
keywords: 'Docker,Hexo,System,Container'
abbrlink: 47633
date: 2017-05-27 16:17:54
---



# Introduction
這篇延續上一篇[Docker image for Hexo (一)](https://blog.hwchiu.com/docker-build-image.html)，要使用上次的概念來打造一個屬於我自己的 **hexo docker image**，至於這邊為什麼是說屬於我自己的？
目前網路上其實也有不少關於 **hexo** 相關的 docker image，在使用上大致上可以分成兩類
- 將 **docker image** 當成一次性的使用，可能提供了 **setup local server** 或者是 **deploy to git** 之類的服務
    - 在這種架構下，通常都是把整個 blog 的 source 放在外面的 **host** 上，再透過 **docker run** 的時候，將這些檔案透過 **volume**的方式掛載到 **container** 內，然後 **container** 內就使用已經安裝好的 **hexo** 環境幫你產生一次性的 **generate**, **deploy** 之類的指令。
    - 這種 **image** 我覺得大概跟我差不多，都是為了練習而產生的，實際上使用沒有特別方便，原因在於 **hexo** 本身就是透過 **npm** 管理了，所有使用到的 **modules** 也都存放在自己本身的資料夾內，這種情況下根本沒有真的幫助使用者減少多少時間，畢竟 **hexo** 本身的安裝就一兩行就結束了。
    - 此外，這種模式最大的麻煩就是，因為你 **hexo** 的安裝都是在 **image**階段就結束了，你若有想要安裝額外的 **npm modules** 就會沒有辦法,所以其實使用上也不夠靈活

<!--more-->
- 第二種就比較偏向將整個 **source**都放進去 **image** 內，然後提供不同的方法讓作者可以編輯文章並且 **generate**/**deploy**。 目前有看過的一種做法是讓他跟 **github**上面的 **webhook** 結合，然後當在 **github**編輯後，就會觸發 **webhook**，驅動 **image**內的 **hexo** 去進行 **generate**以及**deploy**。

在比較兩種類型後，我的使用情境比較會偏向第二種，不過第二種要弄得很完美其實苦工多非常多，否則其實就只是一個撰寫環境的打包 **image**。
不過也沒關係，就當作一個經驗練練 **docker** 也好。
由於我自己的這個專案只有打算給自己用，我主要想要省下的時間有
1. git/zsh 相關環境以及習慣的設定
2. hexo 相關專案的抓取

所以我的 **image** 裡面做的事情大概如下
1. 安裝相關套件(vim/zsh/node)
2. clone blog-source
3. 安裝 hexo
這樣之後就可以進去直接該 container 並且在裡面進行 **generate**/**deploy**。
同時因為我的 hexo 是直接安裝我 blog-source 那一包檔案，因此我只要維護我自己 **npm**相關的檔案，image重新產生的時候裡面放的就會直接更新到最新的那一包 **git repo**了。

不過由於第二點的路徑是綁死在 **image** 內，所以這整個 **docker image**就當作一個自我對於 **docker** 的練習即可。

# Steps

接下來就介紹一下我自己的 **Dockerfile**
在這之前，我有把我常用的操作設定檔**vim/git**有開一個獨立的**git repo**，且我自己本身 **blog** 的 source 也有額外開一個 **git repo**，該repo內就是整個 **hexo** 的檔案，包含了 **npm** 安裝的資訊以及第三方 **theme** 的資訊。

首先，一開始先安裝會用到的套件，這邊包含了 **vim/node/git**。

```bash
RUN apt-get update && \
apt-get install -y git &&\
apt-get install -y vim &&\
apt-get install -y curl &&\
curl -sL https://deb.nodesource.com/setup_6.x | bash - && \
apt-get install -y nodejs  &&\
npm install npm -g  &&\
apt-get clean && \
rm -rf /var/lib/apt/lists/*
```

接者就要先把自己常用的安裝檔案透過 **git** 的方式給抓下來

```bash
WORKDIR /
RUN git clone https://xxxx/xxxxx/config.git && \
        cp config/.gitconfig ~/
```

再來我們就要處理 **hexo**，我希望將 **hexo** 給存放到 **/hexo** 這個路徑上，所以先透過 **WORKDIR** 切換當前位置。
接下來
1. 透過 **git clone** 整個 **hexo source repo**
2. 透過 **npm** 安裝**hexo-cli**
    - 這邊若安裝成全系統的**-g**會遇到一些錯誤，似乎跟當前執行者的身分有關，這邊就沒有花太多時間去研究
    - 之後為了處理這個問題，就把 **hexo-cli/bin** 加入倒當前 **PATH** 即可。
3. 透過 **npm** 安裝相關套件

```bash
WORKDIR /Hexo
RUN git clone --recursive https://xxxx/xxxx/xxx.git && \
        cd blog-source && \
        npm install hexo-cli &&\
        npm install &&\
```

然後處理 **zsh/vim** 的問題。
- zsh 這邊想要使用 **oh-my-zsh**，安裝步驟參考這篇[文章](https://gist.github.com/tsabat/1498393)
- vim 的部分則是將之前上述步驟抓下來設定檔都複製到相關的位置
    - 這邊需要 **vim** 原因是沒有調整過語系的 **vim**沒有辦法順利地開啟中文內容的檔案。

```bash
apt-get update && \
apt-get install -y zsh &&\
apt-get install -y git-core &&\
apt-get install -y wget &&\
apt-get install -y tig &&\
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh || true &&\
chsh -s `which zsh` &&\
echo "export PATH=$PATH:/Hexo/blog-source/node_modules/hexo-cli/bin/" >> ~/.zshrc  &&\
mkdir ~/.vim &&\
cp /config/vim/.vimrc ~/ &&\
cp -r /config/vim/colors ~/.vim
```

最後，為了讓透過 **docker run**執行後，可以直接採用 **zsh** 當作進去的 **shell**，我們要透過 **ENTRYPOINT**去設定進入點，加入下列敘述於 **Dockerfile** 之中。

```bash
ENTRYPOINT ["/usr/bin/zsh"]
```

所以最後整個檔案的樣子大概就是類似

```bash
FROM ubuntu:16.04
MAINTAINER sppsorrg@gmail.com

RUN apt-get update && \
        apt-get install -y git &&\
        apt-get install -y vim &&\
    apt-get install -y curl &&\
    curl -sL https://deb.nodesource.com/setup_6.x | bash - && \
    apt-get install -y nodejs  &&\
    npm install npm -g  &&\
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /
RUN git clone https://xxxx/xxxxx/config.git && \
        cp config/.gitconfig ~/
WORKDIR /Hexo
RUN git clone --recursive https://xxxx/xxxx/xxx.git && \
        cd blog-source && \
        npm install hexo-cli &&\
        npm install &&\
        apt-get update && \
        apt-get install -y zsh &&\
        apt-get install -y git-core &&\
        apt-get install -y wget &&\
        apt-get install -y tig &&\
        wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh || true &&\
        chsh -s `which zsh` &&\
        echo "export PATH=$PATH:/Hexo/blog-source/node_modules/hexo-cli/bin/" >> ~/.zshrc  &&\
        mkdir ~/.vim &&\
        cp /config/vim/.vimrc ~/ &&\
        cp -r /config/vim/colors ~/.vim

#set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936
#set termencoding=utf-8
#set encoding=utf-8

ENTRYPOINT ["/usr/bin/zsh"]
```

# Usage
接下來要使用的話，先透過 **docker build** 產生對應的 **image**，最後使用 **docker run**運行對應的 **container**

```bash
docker build -t hwchiu:test .
docker run --name gg -it hwchiu:test
```

進去該 **container** 後，可以先到 **/Hexo/blog-source** 執行 **hexo g** 確定可以產生文章，且透過 **vim**觀看文章都沒問題後，就透過 **ssh-keygen -t rsa -b 4096 -C "your_email@example.com"** 產生對應的 **key**
接者將上述產生的 **public key**送到 **GITHUB** 去，這樣就可以使用 **hexo deploy** 此功能了。
- 這邊是因為我的 git 是採用 **https**的方式去抓的，所以這邊要去特別設定。

# Summary
經由本次一個簡單的練習，將常用的 **hexo** 的環境給打包起來，雖然不算完美，在編輯文章上還是要透過別的方法先撰寫好，在透過 **vim** 貼到 **hexo** 內來產生跟發布。也許哪天有機會再來嘗試改善，看看是否可以把[hexo-editor](https://www.npmjs.com/package/hexo-editor)整合進去，這樣該 **container** 也可以提供一個外部的網頁服務，直接編輯內部的 **hexo project**。



# 常用指令
- docker images
- docker rm i xxx
- docker stop `docker ps -q -l`
- docker rm `docker ps -q -l`
- docker run -it -name xxx xxx:xxx

