---
layout: post
title: 'FloodlightModule-Topology module'
date: 2013-08-17 13:17
comments: true
categories: [Floodlight, Openflow]
tags:
	- SDN
	- Floodlight
	- Openflow
	- Network
---
Floodlight中，Topology是一個很大的module，牽扯到整個網路拓樸的運算、維護
同時也會維護routing路徑以及broadcast tree的ㄧ些資訊。


##Architecture##

- Cluster.java
- ITopologyListener.java
- ITopologyService.java
- NodePair.java
- NodePortTuple.java
- OrderedNodePair.java
- TopologyInstance.java
- TopologyManager.java
- web

<!--more-->

###Cluster###

*Cluster*
再controller的觀點中，ㄧ個cluster就是一個SCC( strongly connective component)，cluster中的switch都有辦法到達彼此，因此不同cluster間的switch基本上不能互相傳送資料，除非中間經過ㄧ些傳統的switch( controller不知道有傳統switch的存在)

這個檔案定義ㄧ個cluster所需要的基本屬性
```
   protected long id; // the lowest id of the nodes
   protected Map<Long, Set<Link>> links; // set of links connected to a node.
```
- id:該cluster中最小ID的switch dpid.
- links:該cluster中所有的link,每個link包含
	- soruce switch dpid
  - source switch port
  - destination switch dpid
  - destination switch port

###ITopologyListener###
Topology提供的callBack function.

```
void topologyChanged(List<LDUpdate> linkUpdates);
```
當有拓樸發生變化的時候，就會呼叫此function,並且把所有變動的Link (LLDP)都當作參數傳入。

###ITopologyService###
Topology提供的主動API,主要可以讓其他的module來獲取topology上的相關資訊

- **isAttachmentPointPort**  
  根據switch's dpid、switch port 判斷是否有device連結到該swtich的port上面。
  若存在就是回傳false,否則回傳true,不清楚為什麼要反轉。
- **getOpenflowDomainId**  
	根據switch dpid，取得該swtich所屬cluster。
- **getL2DomainId**  
  目前的版本中，做的事情同等於 `getOpenflowDomainId`
- **inSameOpenflowDomain** 
  判斷兩個switch是否屬於同一個cluster。
- **getSwitchesInOpenflowDomain**  
  取得該swtich所屬cluster中的所有switch。
- **inSameL2Domain**  
  目前的版本中，做的事情同等於 `inSameOpenflowDomain`
- **isBroadcastDomainPort**  
  判斷該switch的某個port是否屬於broadcast tree的一部分。
- **isAllowed**  
  不明，總是回傳true。
- **isConsistent**  
  不是很清楚，感覺是判斷一個device的新後位置是否相同。待確認  
  deviceManager.java && Device.java 都有使用到，看起來跟位置有相關。
- **isInSameBroadcastDomain**  
  兩個swtich的port是否屬於同一個broadcast tree。
- **getPortsWithLinks**  
  取得一個switch上的所有ports。
- **getBroadcastPorts**  
  取得一個switch上所有屬於broadcast tree port的port
- **isIncomingBroadcastAllowed**  
  判斷該switch的某個port是否能夠接受broadcast的封包，若該port不屬於broadcast tree就會丟棄該封包。
- **getOutgoingSwitchPort**  
	意義不明
- **getIncomingSwitchPort**  
	意義不明
- **getAllowedOutgoingBroadcastPort**  
	尚未實作，意義不明
- **getAllowedIncomingBroadcastPort**  
	尚未實作，意義不明
- **getBroadcastDomainPorts**  
  取得broadcast tree的所有成員set<dpid,port>
- **getTunnelPorts**   
	取得目前的TopologyInstance中是否是tunnelPorts  
  意義不明
- **getBlockedPorts**   
	尚未實做完成，估計是取得所有被封住的ports。
- **getPorts**    
  取得該switch上所有未被隔離且可以使用的port。

###NodePair###
**(此class目前沒有被使用)**

###OrderdNodePair###
**(此class目前沒有被使用)**


###NodePortTuple###
定義了一個Tuple,包含了一個swtich dpid以及對應的port

``` java
    protected long nodeId; // switch DPID
    protected short portId; // switch port id
```

- **nodeId** :switch dpid
- **portId** :switch port id

###TopologyInstance###
網路拓璞真正的物件，每次同時只會有一個Instance存在，當網路拓樸改變時，就會創造一個新的TopologyInstance，大部分的service都是在這邊真正實作。

TopologyInstance主要分成兩個部分

1. Cluster的相關運算
2. TopologyService的implement

Cluster的運算則透過compute來完成，其中又有四大項目分別去跑。
- compute
	- identifyOpenflowDomains
	- addLinksToOpenflowDomains
	- calculateShortestPathTreeInClusters
	- calculateBroadcastNodePortsInClusters  

**identifyOpenflowDomains**

找出在controller底下的所有cluster,並且把所有的swtich都區分到各自的cluster中。

**addLinksToOpenflowDomains**

把每個cluster中的所有link都記錄起來，以 **Cluster**物件存放所有link。

**calculateShortestPathTreeInClusters**

計算cluster中每個switch到彼此間的最短路徑。

```
for cluster in clusters
	for switch in cluster.getSwitch
  		calculate shortest path for switch in the cluster
```

**calculateBroadcastNodePortsInClusters**
計算每個cluster中的broadcast tree。

實際上就只是取該cluster中最小id switch的shortest path tree 當作該cluster的broadcast domain。


經過這四個步驟後，整個拓樸中 兩兩swtich間的路徑就決定好了，同時broadcast tree也建立完成，所以也不會有broadcast storm的問題。


###TopologyManager###
網路拓樸的管理者
**待續**
