---
title: cscope
keywords: 'cscope,Ubuntu'
date: '2013-10-05 09:25'
comments: true
tags:
  - System
abbrlink: 60103
description:
---

## Introducion
Cscope 是一個用來trace code還滿方便的工具
我通常都用他來trace linuxe kernel code,雖然說有網頁版的reference可以使用，但是用起來不順手，網頁會卡卡的
因此還是習慣使用這種互動式的trace tools
<!--more-->

## Install

`sudo apt-get install cscope` on **Ubuntu**

`portmaster devel/cscope` on **FreeBSd**


### Usage
詳細的可以參考man page. 通常我只有使用 -R 來觀看而已

第一次執行的時候，會花比較久的時間去建立一個**cscope.out**的檔案，會把一些相關資訊放進去


下次執行的時候就會利用該out檔案來作查詢。


### 其他

預設的情況下，cscope只能讀取
- .c
- .h
- .l
- .y

想要讓他讀取java或是cpp的專案，就必須要先自己建置該資料庫
- find ./  -name  *.cpp > cscope.files
- fine ./  -name  *.java >> cscope.files
- cscope -bkq

前面兩行會把所有的檔案路徑都寫入倒cscope.files裡面
- b:建立索引文件
- k:建立索引文件時不會去搜尋/usr/local/目錄
- q:生成cscope.out，加速索引,該檔案包含
  - locate functions
  - function calls
  - macros
  - variables
  - preprocessor symbols

接下來只要使用cscope就可以了
