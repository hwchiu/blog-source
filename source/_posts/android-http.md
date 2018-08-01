---
title: How to download http file in Android
keywords: 'Android, Java, System'
tags:
  - Android
  - Java
  - System
abbrlink: 24263
date: 2017-09-12 08:35:38
description:
---


本篇文章用來記錄如何在**Android**裡面透過 **Http** 抓取檔案

這邊主要會用到兩個物件，分別是 **URL** 以及 **HttpURLConnection**。

步驟如下
1. 根據目標的`URL`去初始化對應的**URL**物件
2. 透過該 **URL** 得到對應的 **HttpURLConnection**
3. 從該 **HttpURLConncetion** 取得回應，譬如 `Header`或是`Body`

<!--more-->

所以接下來看一下每個詳細步驟,這邊假設使用 **http://127.0.0.1/test** 作為檔案的測試

```Java
URL url = null;
try {
    url = new URL("http://127.0.0.1/test");
} exception (MalformedURLException e) {
    System.out.println(e.getMessage());
}

```
由於 **URL** 本身會有 **MalformedURLException** 要處理，所以記得用 **Try/Catch** 包起來處理一下錯誤


```Java
HttpURLConnection httpConn = (HttpURLConnction)url.openConnection();
```
這樣就可以取得該 **HTTP** 的連線了，接下來就可以針對 **ResponseCode** 以及 **Data** 本身去做後續的處理

```Java
int responseCode = httpConn.getResponseCode();

if (HttpURLConnection.HTTP_OK == responseCode) {
    InputStream is = httpConn.getInputStream();
    //Handle InputStream
}
```

