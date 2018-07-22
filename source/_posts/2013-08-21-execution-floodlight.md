---
layout: post
title: 'Execution Floodlight '
date: 2013-08-21 04:39
comments: true
tags:
	- SDN
	- Openflow
	- Network
	- Floodlight
keywords: SDN,Controller,Floodlight
---
記錄一下執行floodlight時，有ㄧ些參數可以使用，都是用來指定設定檔的位置。

###Floodlight configuraion:###
**--configFile ${configuration path}**

###Log configuraion:###
**-Dlogback.configurationFile=${FL_LOGBACK}**

##範例##

`java -Dlogback.configurationFile=logback.xml floodlight.jar --configFile floodlightdefault.properties`
