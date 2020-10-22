---
title: Adapter Pattern
keywords: 'DesignPattern, Adapter Pattern'
date: '2013-11-24 03:53'
comments: true
tags:
  - DesignPattern
abbrlink: 62386
description: Adapter Pattern這個模式是用來讓兩個已經存在但是不相容的介面能夠相容的一種方式。如果資源允許且有辦法，通常都可以直接修改這些已經存在的介面原始碼，擴充讓他能夠支援全新的功能，然而在部分的情況下，其實舊有的介面並不能這樣重新修改。因此這時候可以採用 Adapter Pattern 的方式重新打造一個接口，上承新接口，下承舊接口，藉由這個方式讓舊有的城市不用修改也能夠正常運作。

---

## Introduction


舉例來說，今天有一家廠商開發數位電視A，並有且對應的API可以讓使用者去操縱

於是就有一家廠商根據這個API開發了對應的遙控器B。

<!--more-->

![test.png](http://user-image.logdown.io/user/415/blog/415/post/161782/joRQRvslT8Kyo1xKe94q_test.png)

這時候一切都很正常work,但是不久之後，該廠商又開發了一台更新型的電視C，

這時候提供的API卻跟原來的完全不一樣，這時候原本的遙控器就完全沒有辦法去操控這台新的電視C

這時候解決方法如下

- 重新製作一個新的遙控器，然後這個新的遙控器可以支援新舊兩款電視。
- 對新的電視B製造一個轉接器，能夠再新舊API運作，使得舊有的遙控器能夠順利使用。


![test.png](http://user-image.logdown.io/user/415/blog/415/post/161782/tW7GgPSTC2mJAahsN7Qi_test.png)

如果重新製作一個遙控器，每次有新的API出現，就要重新改寫遙控器，此外在維護上面也複雜。

因此這邊採用轉接器的方式製作。



![test.png](http://user-image.logdown.io/user/415/blog/415/post/161782/0OwKO5u0SqSz3YH1szpF_test.png)

再智慧電視B的前面多一層轉接器，這轉接器提供舊有一致的API給搖控器使用，底層使用新的API與電視B溝通，如此一來
遙控器本身依然可以正常運作的去操控新舊兩款電視。



## Example


``` java

//television A

public class TelevisionA implements Television{
	public old_control{ .... }
	....
}

//television C
public class TelevisionB{
	public new_control{ ...}
  ....
}

//television adapter
public class TVBdapter implements Television{
  	TelevisionB tb;
	public old_control{   tb.new_control() ....}
  ...
}
//controller
public class controller{
	televisionList = {TelevisionA, TVBdapter };
	for(Television tv : televisionList){
		tv.old_control(...)
	}
	....
}


```

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/mLwKb3zrx

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4XNG

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