---
layout: post
title: windows_vpn
date: '2013-03-29 15:02'
comments: true
tags:
  - System
  - Windows
abbrlink: 13639
---

最近因為某個教授的要求，希望windows開機就可以自動vpn連線，所以這部份花了一些時間去研究，雖然我認為每次開機自己動手點兩下好像也沒有多困難阿~冏
<!--more-->

這個概念其實不難，寫一個可以連線的batch file,每次開機的時候，自動去執行該batch file，就可以達到連線的功能了。

1. 在網際網路那邊手動增加一個VPN連線，假設該VPN連線名稱為 vpn_connection。

2. 寫一個batch file,內容增加一行

>rasdial "my_vpn_connection" "myname"  "mypasswd"

這時候可以手動執行看看，看會不會連線成功，如果連線不會成功，就根據錯誤代碼去解決。

3. 執行taskschd.msc 這個排班程式，把該batch file加入至開機執行，並且在網路連線成功後才執行。

重開機測試!

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/JPwq4znr1

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=7EP1KY3

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