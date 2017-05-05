---
layout: post
title: 'X Window  & X WM on FreeBSD 9.1R'
date: 2013-10-05 09:04
comments: true
categories: [System, FeeBSD]
tags:
	- System
	- FreeBSD
---

##X Window##
X window 是一個再Unix-base system的GUI Desktop.
最初設計的目的就是希望是一個應用程式，所以要盡量的跟硬體沒有關係
架構中分成**client**跟**server**. 彼此透過網路傳送資料，

1. 一個server可以跟很多個client連接
2. server端要接螢幕、鍵盤、滑鼠。使用者使用 server來操控
3. server會把一些in/out的event通知每個client，每個client各自運算完畢後，再告訴server要如何把畫面給渲染出來
4. 整個運算主要集中在client身上，client是個應用程式，可以是個瀏覽器，可以是個播放器...等

<!--more-->
 
最簡單的架構下，client跟server放在同一台電腦中，然後直接很多個client，所有的client組合而成一個GUI 

###Install###

- portmaster  x11/xorg
- 安裝 滑鼠跟鍵盤的驅動
  - sysutils/hal
  - devel/dbus
  - hald_enable="YES" >> /etc/rc.conf
  - dbus_enable="YES" >> /etc/rc.conf

###Config###
- Xorg -configure  (產生 X11預設設定檔)
- 測試設定檔OK與否
	- Xorg -config /root/xorg.conf.new 
- cp /root/xorg.conf.new /etc/X11/xorg.conf
如果有要針對一些硬體、顯示、滑鼠去作調整，就針對這個xorg.conf去編輯即可

###Run###
- startx
	- 這時候會看到三個視窗，就代表X11安裝成功了

##X Window Manager##
Window Manager(WM) 可以看作一個特別的X client, 提供了類似windows的介面給使用者使用，
	- 背景、主題、桌布
  - 虛擬桌面
  - 視窗特性
  	- 移動、放大、縮小...

再X server跟X client之間的溝通都會被導到WM來處理。

比較之名的有

- Gnome
- KDE
- XFCE
- ....等
   

###Install###

- Xfce
	- x11-wm/xfce
- KDE
	- x11/kde4

###Config###
接下來要編輯xinitrc檔案，讓我們執行Xorg的時候會去執行WM
- 預設的檔案位置 `/usr/local/lib/X11/xinit/xinitrc`
- 每個帳號的位置 `~/.xinitrc`
- `echo "/usr/local/bin/startkde4" > ~/.xinitrc`
如此一來，當執行startx的時候，會先讀取家目錄底下的.xinitrc,然後就去執行對應的WM

###Run###
- startx
