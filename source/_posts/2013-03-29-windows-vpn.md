---
layout: post
title: 'windows_vpn'
date: 2013-03-29 15:02
comments: true
tags:
	- System
	- Windows
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
