---
layout: post
title: Install News server on FreeBSD 9.1R
date: '2013-10-05 09:02'
comments: true
tags:
  - System
  - FreeBSD
abbrlink: 65511
---
這邊整理一下安裝csnews2時遇到的一些問題，並且筆記一些操作

###安裝 & 設定 ###
參考CC wiki
<!--more-->

###文章轉移###

- rsync cycbuff
- rsync db/history
- 重新建立overview
	- ctlinnd pause 'make overview'
	- makehistory -x -O -b   
  	 x: won't write out history file entries.  
     O: Create the overview database  
     b: Delete any messages found in the spool that do not have valid Message-ID: headers in them. 
   - makedbz -i  
     i:To ignore the old database  
  - ctlinnd go 'over'

###設定檔檢查###

1. inncheck  (inn.conf)
2. scanspool -v (active, spool)

###更新相關設定###

- 重新編譯innd,進入innd src底下
- ./configure --opetions
- make && make update

###創新的newsgroup###

- ctlinnd newgroup name
- modity db/newsgroup


###其他###

1. 創新newsgorup


1. 執行innd & nnrpd 會噴權限不足
	 - 檢查/news/bin/innbind 有無SUID
