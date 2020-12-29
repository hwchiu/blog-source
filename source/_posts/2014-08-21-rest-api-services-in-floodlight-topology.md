---
layout: post
title: REST API services in Floodlight (Topology)
date: '2014-08-21 04:34'
comments: true
tags:
  - SDN
  - Openflow
  - Java
  - Network
  - Floodlight
  - SourceCode
keywords: 'SDN,Controller,Floodlight'
abbrlink: 65061
---
Introduction
---------
1. Topology這邊則是維護整個網路拓樸的資訊，包含**Link**、**Cluster**以及**routing path**。
2. Topology的所有資訊都是建立於**LinkDiscovery**所發送的**LLDP**與**BDDP**，再搭配演算法去得到整個網路拓樸的情況。
3. 每個Cluster都會包含一個以上的Switch，同一個Cluster內的Switch可以組成一個Strongly Connected Components (SCC)。


<!--more-->


API
------
- **/wm/topology/links/json**  : 回傳所有Links，盡可能回傳**BIDRECTONAL**
  1. Method: **GET**
  2. Parameter: 無，
	3. Code:
	    + TopologyWebRoutable.java
      + LinksResource.java
	4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/links/json | python -mjson.tool

```json
[
    {
        "direction": "bidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:04",
        "src-port": 3,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    },
    {
        "direction": "bidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:05",
        "src-port": 4,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    },
    {
        "direction": "bidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:02",
        "src-port": 1,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    },
    {
        "direction": "bidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:03",
        "src-port": 2,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    }
]
```


- **/wm/topology/directed-links/json** : 回傳所有Type是 **DIRECT_LINK**或是**TUNNEL**的Link，都以**UNIDIRECTIONAL**的方式呈現
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有符合條件的Links
	3. Code:
	    + TopologyWebRoutable.java
      + DirectedLinksResource.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/directed-links/json | python -mjson.tool

```json
[
    {
        "direction": "unidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:05",
        "src-port": 4,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 3,
        "dst-switch": "00:00:00:00:00:00:00:01",
        "src-port": 1,
        "src-switch": "00:00:00:00:00:00:00:04",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:01",
        "src-port": 1,
        "src-switch": "00:00:00:00:00:00:00:02",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:03",
        "src-port": 2,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 2,
        "dst-switch": "00:00:00:00:00:00:00:01",
        "src-port": 1,
        "src-switch": "00:00:00:00:00:00:00:03",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 4,
        "dst-switch": "00:00:00:00:00:00:00:01",
        "src-port": 1,
        "src-switch": "00:00:00:00:00:00:00:05",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:04",
        "src-port": 3,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    },
    {
        "direction": "unidirectional",
        "dst-port": 1,
        "dst-switch": "00:00:00:00:00:00:00:02",
        "src-port": 1,
        "src-switch": "00:00:00:00:00:00:00:01",
        "type": "internal"
    }
]

```
- **/wm/topology/external-links/json** :  回傳透過BDDP所發現的Links。External-Links可以用來找出在網路環境中，不受controoler控制的switch的links。使用情境請參考[這篇](http://rascov.logdown.com/posts/183374-detect-legacy-switches-in-sdn-environment-using-floodlight)
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有符合條件的Links
	3. Code:
      + TopologyWebRoutable.java
	    + ExternalLinksResource.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/external-links/json | python -mjson.tool

```json
[
    {
        "src-switch": "00:00:00:00:00:00:00:02",
        "src-port": 3,
        "dst-switch": "00:00:00:00:00:00:00:03",
        "dst-port": 3,
        "type": "external",
        "direction": "bidirectional"
    }
]
```
- **/wm/topology/tunnellinks/json** :    回傳所有的tunnelLink,我還沒有嘗試過有tunnel的網路拓墣，所以此API也不是很了解，只能從code去看。
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有tunnelLinks，每條link都會以**NodePortTuple**的形式呈現，該型態包含了switch的DPID以及對應的Port
	3. Code:
	    + TopologyWebRoutable.java
      + TunnelLinksResource.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/tunnellinks-links/json | python -mjson.tool
- **/wm/topology/switchclusters/json** :  回傳Controller底下的所有**cluster**，每個**cluster**是由一群switches所組成的SCC。
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有的cluster
	3. Code:
      + SwitchClustersResource.java
	    + TopologyWebRoutable.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/switchclusters/json | python -mjson.tool

```json
{
    "00:00:00:00:00:00:00:01": [
        "00:00:00:00:00:00:00:01",
        "00:00:00:00:00:00:00:02",
        "00:00:00:00:00:00:00:03",
        "00:00:00:00:00:00:00:04",
        "00:00:00:00:00:00:00:05"
    ]
}
```
由此範例可以看到目前網路拓墣中有一個cluster，包含了五個switches。cluster的識別碼就由最小的switch DPID來決定。由於是SCC的關係，這五個switches都有辦法傳送封包到彼此。

- **/wm/topology/broadcastdomainports/json** : 回傳所有的broadcast domain ports，此port跟externel link有關係。返回值代表某switch上的某port所連接到的是一個不受controller所控制的switch。所以未來若是有收到廣播封包的話，這邊的要收起來。
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有符合條件的**NodePortTuple**
	3. Code:
	    + TopologyWebRoutable.java
      + BroadcastDomainPortsResource.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/broadcastdomainports/json | python -mjson.tool

```json
[
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:05"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:04"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:03"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:02"
    }
]
```
- **/wm/topology/enabledports/json** :  回傳所有Switch上面的所有Ports。
	1. Method: **GET**
	2. Parameter: 沒有參數
	3. Code:
      + EnabledPortsResource.java
	    + TopologyWebRoutable.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/enabledports/json | python -mjson.tool

```json
[
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:02"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:02"
    },
    {
        "port": 65534,
        "switch": "00:00:00:00:00:00:00:02"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:03"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:03"
    },
    {
        "port": 65534,
        "switch": "00:00:00:00:00:00:00:03"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:04"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:04"
    },
    {
        "port": 65534,
        "switch": "00:00:00:00:00:00:00:04"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:05"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:05"
    },
    {
        "port": 65534,
        "switch": "00:00:00:00:00:00:00:05"
    }
]
```
- **/wm/topology/blockedports/json** : 此API我還沒有測試成功，看CODE也沒有發現情況的port會被加入到blockedports，會再寄信詢問。
	1. Method: **GET**
	2. Parameter: 沒有參數，就回傳所有符合條件的Links
	3. Code:
	    + TopologyWebRoutable.java
      + BlockedPortsResource.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/blockedports/json | python -mjson.tool
- **/mw/topology/route/{src-dpid}/{src-port}/{dst-dpid}/{dst-port/json** : 一個用來找出最短路徑的REST API
	1. Method: **GET**
	2. Parameter:
  		+ src-dpid: source switch dpid
      + src-port: source port of the source switch
      + dst-dpid: destination switch dpid
      + dst-port: destination port of the destination switch
	3. Code:
	    + TopologyWebRoutable.java
      + RouteResource.java
  4. Example:
  		+ curl -s http://127.0.0.1:8080/wm/topology/route/00:00:00:00:00:00:00:04/2/00:00:00:00:00:00:00:07/2/json | python -mjson.tool

```json
[
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:04"
    },
    {
        "port": 3,
        "switch": "00:00:00:00:00:00:00:04"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:02"
    },
    {
        "port": 3,
        "switch": "00:00:00:00:00:00:00:02"
    },
    {
        "port": 1,
        "switch": "00:00:00:00:00:00:00:01"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:01"
    },
    {
        "port": 3,
        "switch": "00:00:00:00:00:00:00:05"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:05"
    },
    {
        "port": 3,
        "switch": "00:00:00:00:00:00:00:07"
    },
    {
        "port": 2,
        "switch": "00:00:00:00:00:00:00:07"
    }
]
```
以一個 --topo tree,3的範例來說，若想要從switch4的port2到switch7的port2，可以得到上述結果的走法。
