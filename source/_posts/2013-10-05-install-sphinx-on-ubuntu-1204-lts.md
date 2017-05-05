---
layout: post
title: 'Install Sphinx on Ubuntu 12.04 LTS'
date: 2013-10-05 09:01
comments: true
categories: [System, ubuntu, Sphinx]
---
##Sphinx##

Sphinx是一套建置說明文建的軟體，本身是用python寫成的
目前使用Sphinx這套軟體來當作會議紀錄

http://nslab.cs.nctu.edu.tw/sphinx/NSLMeeting/

<!--more-->

###Install###

直接透過atp-get 安裝即可

`sudo apt-get install sphinx`


###Config###

安裝完畢後，執行

`sphinx-quickstart`就可以基本設定了

每個選項都有說明，基本上都採用預設值即可

- 設定檔: conf.py

	-  外掛管理
  -  資料夾結構管理
  -  一些通用參數，如作者名稱，版本...等
  
- 主要的檔案: index.rst
	-. 檔案的結構
  -. toctree

###index.rst###
```
Lab Meetgins
=============
.. toctree::
   :maxdepth: 4
   :titlesonly:

   20130924.rst
   20131001.rst

國科會 meetings
===============
.. toctree::
   :maxdepth: 4
   :titlesonly:

   20130925.rst
```

這邊我定義兩個toctree，每個toctree底下又會有其他的rst，結構大概是這樣

- Lab Meetings
	- 20130924.rst
  - 20131001.rst
- 國科會 meetings
	- 20130925.rst
  
總共兩個分類，每個分類底下的文章都是一個額外的rst檔案

在toctree底下的都是一些設定參數

- maxdepth : 最大深度
- titlesonly : 在首頁面只顯示子類的標題


###Write###

Sphinx採用的`reStructuredText`
格式跟markdown很類似，但是複雜了一些
官方網站有滿詳細的介紹，有需要時再去參考即可


###Build###

如果想要轉成html網頁，有兩種方法可以執行

1. sphinx-build -b html .  NSLMeeting
 	 意思是建置html的網頁， 然後以當前目錄為source 來源，然後把檔案build到NSLMetting去。  
   
2. make html
   在Makefile中定義了相關得動作，當執行`make html`的時候，其實就是執行
   `sphinx-build -b html . _build/html` 

這邊因為我想要直接弄到別的資料夾，所以我直接設定aliase去執行方法1

目前對於這套軟體還在學習階段，有任何學習會繼續紀錄。


