---
layout: post
title: 'REST API services in Floodlight (Device)'
date: 2014-08-20 17:13
comments: true
categories: 
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
---------
1. Device的API主要分成兩類，回傳符合條件的**Device**或是**Entity**
2. Entity是一個包含**MAC**、**VLAN**、**IPV4**型態，用來代表網路中最基本的一個元件
3. Device可以包含很多個Entity，每個Device除了會有多個Entity外，還會有所謂的AttachmentPoint。AttachmentPoint會紀錄該Device是與哪個Switch的哪個Port相連
4. 一般情況下， Device與Entity是1:1的關係，但是某些情況下，可能一個Device會擁有多個AttachmentPoint、多個IP address或是相同的MAC address擁有不同的VLAN TAG，在此情況下就會有一個Device擁有多個Entity。

<!--more-->


API
------
- **/wm/device/**  : 回傳Devices
  1. Method: **GET**
  2. Parameter: **這邊都是過慮的條件，只有符合這些條件的device才會被選出來。如果什麼參數都沒有，預設就是所有device**。
      + mac :  device的mac address
      + vlan : device的vlan tag
      + ipv4 : device的ipv4 address
      + dpid : 這個device所連接到的switch的dpid
      + port : 這個device是連接到該switch的哪個port
      + mac_startwith : 以下的參數都如同上面的概念，只不過上面的是要完整符合，這邊的是開頭符合就好
      + vlan_startwith : 
      + ipv4_startwith :
      + dpid_startwith :
      + port_startwith :
	3. Code:  
      + DeviceRoutable.java  
      + AbstractDeviceResource.java  
      + DeviceResource.java  
      + DeviceSerializer.java
	4. Example:    
  		+ curl -s http://localhost:8080/wm/device/?ipv4_startwith=10 | python -mjson.tool
      + curl -s http://localhost:8080/wm/device/| python -mjson.tool
      + curl -s http://localhost:8080/wm/device/?ipv4=10.0.0.2  | python -mjson.tool
      
```python
[
    {
        "attachmentPoint": [
            {
                "errorStatus": null,
                "port": 2,
                "switchDPID": "00:00:00:00:00:00:00:03"
            }
        ],
        "entityClass": "DefaultEntityClass",
        "ipv4": [
            "10.0.0.2"
        ],
        "lastSeen": 1408555008093,
        "mac": [
            "00:00:00:00:00:02"
        ],
        "vlan": []
    }
]     
```      
- **/wm/device/debug** : 回傳Entities
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有的Entity。
	3. Code:   
		+ DeviceRoutable.java  
		+ AbstractDeviceResource.jave  
		+ DeviceEntityResource.java    
  4. Example:
  		+ curl -s http://localhost:8080/wm/device/debug | python -mjson.tool
      
```python
[
		[
          {
              "activeSince": 1408555008106,
              "ipv4Address": null,
              "lastSeenTimestamp": 1408555008106,
              "macAddress": "00:00:00:00:00:01",
              "switchDPID": "00:00:00:00:00:00:00:02",
              "switchPort": 2,
              "vlan": null
          },
          {
              "activeSince": 1408555007885,
              "ipv4Address": "10.0.0.1",
              "lastSeenTimestamp": 1408555007885,
              "macAddress": "00:00:00:00:00:01",
              "switchDPID": "00:00:00:00:00:00:00:02",
              "switchPort": 2,
              "vlan": null
          }
      ],
      [
          {
              "activeSince": 1408555008093,
              "ipv4Address": "10.0.0.2",
              "lastSeenTimestamp": 1408555008093,
              "macAddress": "00:00:00:00:00:02",
              "switchDPID": "00:00:00:00:00:00:00:03",
              "switchPort": 2,
              "vlan": null
          }
      ]
]
```

