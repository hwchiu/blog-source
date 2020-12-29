---
title: Sublime Text 3 + cscope (windows)
keywords: 'sublime,sublime3,cscope'
date: '2014-01-08 09:02'
comments: true
tags:
  - System
  - Windows
abbrlink: 57703
description: This post shows a way to install the cscope into your sublime text3 and then you can use the cscope for your existing programming project.

---

## Install Package Control
- 按下 **CTRL+`**
- 貼上
```
import urllib.request,os,hashlib; h = '7183a2d3e96f11eeadd761d777e62404e330c659d4bb41d3bdf022e94cab3cd0'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://sublime.wbond.net/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

Refer to [installation_Package_Control](https://sublime.wbond.net/installation)

## Install cscope

- 按下 **CTRL+SHIFT+P**
- 先輸入 **INSTALL PACKAGE** ，之後再輸入 **cscope**，即可安裝完成
- 產生 **cscope.out**
	* **F:\ovs>cscope.exe -Rbv**

![螢幕截圖 2014-01-08 17.14.03.png](http://user-image.logdown.io/user/415/blog/415/post/174922/RtLdJjoVQGW7bDDISxCe_%E8%9E%A2%E5%B9%95%E6%88%AA%E5%9C%96%202014-01-08%2017.14.03.png)

Refer to [cscope  plugin source](https://github.com/ameyp/CscopeSublime)
## Usage

- 按下 **ctrl+\**後，根據需求來使用
- 直接使用 鍵盤操作
- Ctrl/Super + \ - Show Cscope options
- Ctrl/Super + LCtrl/Super + S - Look up symbol under cursor
- Ctrl/Super + LCtrl/Super + D - Look up definition under cursor
- Ctrl/Super + LCtrl/Super + E - Look up functions called by the function under the cursor
- Ctrl/Super + LCtrl/Super + R - Look up functions calling the function under the cursor
- Ctrl/Super + Shift + [ - Jump back
- Ctrl/Super + Shift + ] - Jump forward




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
