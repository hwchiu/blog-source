---
layout: post
title: 'Freebsd_Quota'
date: 2013-04-07 17:56
comments: true
categories: [System]
tags:
	- System
---

在系統管理時，由於是一個檔案系統給眾多使用者使用，所以為了避免有使用者獨佔整個系統空間，便會對每個使用者設定空間限制。
此外，也可以針對group去設定空間大小，這樣就可以達到分級制度的管理。

在quota管理中，主要是分成三個主要屬性

- **soft limit**: 軟限制，使用者的容量可以超過這個限制，但是在grace period期限內，要將自己的容量給降低到soft limit以下，否則就會無法繼續操作檔案系統。
- **hard limit**: 硬限制: 使用者的容量完全不能超過這個限制。
- **grace period**: 當使用者的容量超過軟限制時，這個時間就會被啟動，使用者要在時間內將自己的容量給降低。
<!--more-->

接下來就來實際在FREEBSD中操作看看。

###Setup Disk Quota in FreeBSD###
####Build Kernel####
因為預設的kernel中並沒有支援這個功能，所以要自己重編kernel,加入
options QUOTA

關於build kernel，參考[這裡](http://www.freebsd.org/doc/handbook/kernelconfig-building.html)

####Edit /etc/fstab####
修改/etc/fstab,對想要進行quota控制的FS進行參數調整

	Device  MountPoint FSType Options Dump Pass  
	/dev/da0p2     /  UFS rw,userquota,groupquota 1 1

接者重新開機，或是remount FS，使其重新讀取設定
####對使用者或是群組 調整其上限####
這邊使用**edquota**這個指令來調整
 

-u: 加上要調整的使用者  
-g: 加上要調整的群組  
-t: 調整grace period  

執行後會看到已EDITOR對應的文字編輯器開啟編輯，會出現類似下面  
>Quotas for user hwchiu:  
>/usr : in use: 11216k, limits (soft = 0k, hard = 0k)
>	inodes in use : 903, limits (soft, hard=0)  

這邊就可以去調整軟硬限制，根據FILE SIZE或是INODES的數量

####**啟動quotacheck**####
使用quotacheck來掃描使用者的使用狀況  
-a : 掃描/ect/fstab底下所有FS中檔案的使用情況  
-v : 詳細過程  
-u : 掃描使用者的檔案情況  
-g : 掃描群組的檔案情況  
 
就給他執行 **quotacheck -avug**

####**啟動quota**####
執行quotaon -a，執行quota限制的功能，沒有開啟的話，一切的設定就只是擺好看的

####**觀看**####
使用**quota**這個指令來觀看  
quota:
-u:使用者名稱  
-g:群組名稱  
-v:詳細  
-h:以容易辨識的格式表達大小，如M、  

>Filesystem        usage    quota   limit   grace  files   quota  limit   grace
>/amd/gcs           305M     390M    410M           6414   40000  42000
>/amd/mail           41M      97M    117M              1       2      3

###Reference###
[FreeBSD Handbook](http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/quotas.html)
