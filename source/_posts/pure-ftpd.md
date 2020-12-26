---
title: 'Install the FTP server on FreeBSD'
keywords: 'FreeBSD,pure-ftpd,ftpd,ftpserver'
date: '2013-10-12 09:03'
comments: true
tags:
  - System
  - FTP
  - FreeBSD
abbrlink: 28373
description: 這邊要跟大家分享的是自架 FTP Server 的一些設定與心得，以往在 Windows 時可以很快速的使用 FilwZilla 來架設 FTP Server, 然而在 FreeBSD 的系統中，我們有哪些相關的選擇可以使用? 本篇文章會採用 Pure-Ftpd 作為一個 FTP Server 並且記錄 FTP 常見的相關用法要如何透過 Pure-Ftpd 來設定

---

在`Windows`上安裝Ftp-server時，通常都是選擇[Filezilla](https://filezilla-project.org/)這套軟體來使用。
透過ＵＩ界面的引導，在安裝以及設定上都非常容易，但是有時候一些比較特殊的要求，未必能夠很好地達成。
在FreeBSD上，安裝系統時有最基本的`ftpd`，或是ports內的`ProFTPD`以及`Pure-ftpd`都能夠用來架設ftp-server.
但是因為FTP 走的是port 21,在預設沒有更改的情況下，這些ftp-server都會嘗試去bind port 21因此同時間只能有一個server在運行。

在System Administration的課程中
第二個作業就要求架設一個ftp-server,並且滿足下列要求，在架設的過程中就順便記錄一下各種設定。
<!--more-->

匿名帳號登入：

- chrooted (/home/ftp)
- Only download from /home/ftp/public
- Can upload & mkdir but no download or delete from /home/ftp/upload
	- Can't download the files upload by anonymous account
  - Can download the files upload by others
- Hidden directory /home/ftp/hidden
	- There is a directory called "treasure" inside
  - Client can't list in /home/ftp/hidden but can in /home/ftp/hidden/treasure

Virtual users:

- name: ftp-vip
- full access on /home/ftp/{public,hidden,upload}
- chhrooted (/home/ftp)
- Hidden directory is visible to ftp-vip
- Only login from some ip

Other:

- Can login with TLS

系統:   **FreeBSD 9.2**
Port:  **Pure-ftpd**

# 前置

- install ports `portmaster ftp/pure-ftpd` (TLS 打勾)
- `echo 'pureftpd_enable="YES"' >> /etc/rc.conf`
- create directory
	- `mkdir -p /home/ftp/public /home/ftp/upload /home/hidden`

# 匿名
- config   `/usr/local/etc/pure-ftpd.conf`
	- NoAnonymous                 no
	- AntiWarez                  yes  (上傳檔案owner是'ftp'的不能刪除)
	- AnonymousCanCreateDirs		 yes
  - AnonymousCantUpload         no
- Add a ftp account for Anonymous
	- `pw groupadd ftpuser`
	- `pw useradd ftp -g ftpuser -d /home/ftp`

# Virtual user
- config `/usr/local/etc/pure-ftpd.conf`
	- PureDB     /usr/local/etc/pureftpd.pdb
- Add a real account
	- pw groupadd virtualgroup
	- pw useradd ftpuser -g virtualgroup -c "FTP visual user" -d /home/ftp -s /sbin/nologin
- Map a virtual account to a real account
	- pure-pw useradd ftp-vip -u ftpuser -g virtualgroup -d /home/ftp -m
- IP limitation.
	- pure-pw usermod ftp-vip -r [IP/mask]   (-r means  allow client's ip)

# Directory permission
## public
pulbic中，讓匿名帳號變成other的權限，然後把w權限給拔掉，這樣對於目錄中有任何異動的行為(刪除、移動、改名)都無法使用。
讓virtualgroup的人也有完整的權限去處理，這樣ftp-vip就有完整權限。

- `chown root:virtualgroup /home/ftp/public`
- `chmod 775 /home/ftp/public`

## upload
upload中，匿名帳號要可以下載跟創立資料夾，以及下載非`ftp`擁有的檔案。
由於先前有設定`AntiWarez`，因此檔案擁有者是`ftp`的就會無法下載，
ftp-vip是group的權限，因此什麼都可以做。
給予其`w`的權限，這樣才可以創立資料夾,然後匿名帳號天生就不可以刪除文件。
- `chown ftp:virtualgroup /home/ftp/upload`
- `chmod 775 /home/ftp/upload`


## hidden
由於目錄的`r`代表的能否看到這些檔案即ls指令，而`x`代表可否進入該資料夾即cd。
因此我們把r拔掉即可達成。

- `chown root:virtualgroup /home/ftp/hidden`
- `chmod 775 /home/ftp/hidden`


# TLS
詳細參考 `/usr/local/share/doc/pure-ftpd/README.TLS`
- 安裝的時候要勾選`TLS`
- config `/usr/local/etc/pure-ftpd.conf`
	- TLS  (0,1,2)
  - 0: 不支援加密傳輸
  - 1: 加密、不加密都支援傳輸
  - 2: 不支援非加密傳輸
  - 選擇2的話，就一定要用ftpes才能連線，選擇1的話，使用ftp or ftpes都可以連線
- create a self-signed certificate
	- 預設的憑證位置是 `/etc/ssl/private/pure-ftpd.pem`
  - 編譯的時候可以透過`make configure CERTFILE=your pem location`來修改位置
	- `mkdir -p /etc/ssl/private`
  - `openssl req -x509 -nodes -newkey rsa:1024 -keyout /etc/ssl/private/pure-ftpd.pem -out /etc/ssl/private/pure-ftpd.pem`
  - `chmod 600 /etc/ssl/private/*.pem`

# Restart pure-ftpd
- /usr/local/etc/rc.d/pure-ftpd restart

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