---
layout: post
title: 'DebugCounter in Floodlight'
date: 2014-08-16 02:10
comments: true
tags:
	- SDN
	- Openflow
	- Floodlight
	- Java
	- Network
	- SourceCode
keywords: SDN,Controller,Floodlight
---
Introduction
-------------
- DebugCounter是一個Floodlight本身就有提供的module，功能非常的簡單，就如同其名一樣，做一個counter，供debug的時候使用

<!--more-->

- 實際上，自己要寫一個counter在module中是非常容易的事情，那為什麼還需要使用DebugCounter來處理? 唯一的好處就是可以透過DebugCounter所提供的REST API將此Counter的資訊給暴露出去，讓外面的應用程式可以透過REST API來存取。這樣的話就不需要自己寫一個REST API來處理了。
- 是採用Hierarchy的架構來存放的， modules/level1/level2/level3, ex: **mymodule/10.0.0.1/Pkt/In**，最多只支援到底下三層的紀錄
- 本身會使用thread-local counter來紀錄，可透過定期或是手動的方式將這些thread-local counter給整合到一個global counter中。



Implementation
---------
- **IDebugCounterService.java** 此檔案定義DebugCounter所需功能的界面
- **IDebugCounter.java** 則定義了每個counter應該要有的功能，如counter++
- **DebugCounter.java** 則實現了IDebugCounterService與IDebugCounter的功能
- **web/** 這邊的檔案則是實現了RESTAPI的處理

Usage
-----
- 先透過Floodlight modules system取得**IDebugCounterService**的instance。
- 呼叫**IDebugCounterService**的registerCounter來註冊，並會回傳一個**IDebugCounter**的物件，之後都透過此物件來進行counter的處理
- IDebugCounter.updateCounterNoFlush()就可以將該counter給遞增，並且不馬上寫回global counter，會較有效率。


Examle
------
- 先取得 **IDebugCounterService**的instance。
``` java
protected IDebugCounterService debugCounters;
this.debugCounters = fmc.getServiceImpl(IDebugCounterService.class);

```
- 宣告一個**IDebugCounter**，並且註冊
``` java
public IDebugCounter cntIncoming;
            cntIncoming = debugCounters.registerCounter(PACKAGE, "incoming",
                "All incoming packets seen by this module", CounterType.ALWAYS_COUNT);
```

- 根據情況來將該counter給+1，以此counter是希望紀錄所有incoming packets，所以收到PacketIn的時候就去遞增該counter
``` java
public Command receive(IOFSwitch sw, OFMessage msg,
  FloodlightContext cntx) {
 	 switch (msg.getType()) {
  		case PACKET_IN:
	  	cntIncoming.updateCounterNoFlush();
```
