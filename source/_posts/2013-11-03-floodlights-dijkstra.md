---
layout: post
title: 'Floodlight Dijkstra'
date: 2013-11-03 09:52
comments: true
categories: [Floodlight, java, Openflow]
tags:
	- Floodlight
	- SDN
	- Java
	- Network
	- Algorithm
---
再 **Floodlight** 中，會定期送出 **LLDP** 的封包去學習當前拓樸的情況
一旦發線拓樸情況有所改變，就會產生一個新的TopologyInstance物件
在這個物件之中就會重新去計算 **broadcast tree** 以及拓樸中每個switch的 **shortest path tree**。

再 **TopologyInstance** 裡面是採用 **dijkstra**的方式來建所謂的routing path.

再 `calculateShortestPathTreeInClusters`裡面
會針對每個cluster中的每個node都去跑一次dijkstra,來建立這個node再該cluster中的shortest path tree.
<!--more-->

####Function####

`protected BroadcastTree dijkstra(Cluster c, Long root,Map<Link, Integer> linkCost,boolean isDstRooted)`

- **Cluster c**: 該node所屬的cluster
- **Long root**: 該node
- **Map<Link, Integer> linkCost**: 這個cluster中所有link的cost,預設中是空的，只有tunnal port對應的link才有cost
- **boolean isDstRooted**: 用來指示 一條link要看其src switch還是dst switch,目前是用true,但是我覺得改成false也不影響結果。

####Member####

-  HashMap<Long, Link> nexthoplinks   
   用來記錄其shortest path tree的結構，key是switch node, value是連接到該switch node是透過哪條link。
-  HashMap<Long, Integer> cost
   用來記錄目前到某個switch node的cost是多少。
-  HashMap<Long, Boolean> seen
   用來記錄某個switch是否已經拜訪過
-  PriorityQueue<NodeDist> nodeq
   一個優先佇列，會根據到達該switch node的cost為基準去排序。
   
 
``` java
   protected class NodeDist implements Comparable<NodeDist> 
   ....
   @Override
   public int compareTo(NodeDist o) {
   	if (o.dist == this.dist) {
  		return (int)(this.node - o.node)
    }
    return this.dist - o.dist;
   }
```
   

###Algorithm###


####Step1####
- 初始化相關容器
- 由cluster取得所有的link，先設定所有switch node的cost都是無限大
- root該switch node的cost是0
- 把root加入到queue內。



``` java

HashMap<Long, Link> nexthoplinks = new HashMap<Long, Link>();
HashMap<Long, Integer> cost = new HashMap<Long, Integer>();
int w;
for (Long node: c.links.keySet()) {
  nexthoplinks.put(node, null);
  cost.put(node, MAX_PATH_WEIGHT);
}

HashMap<Long, Boolean> seen = new HashMap<Long, Boolean>();
PriorityQueue<NodeDist> nodeq = new PriorityQueue<NodeDist>();
nodeq.add(new NodeDist(root, 0));
cost.put(root, 0);

```

####Step 2 ####
- 從queue裡面拿出cost最小的node
- 取得到達該node的cost
- 做個錯誤檢查
- 如果該node已經檢查過了，就忽略。
- 把該node加入到seen裡面


``` java
        while (nodeq.peek() != null) {
            NodeDist n = nodeq.poll();
            Long cnode = n.getNode();
            int cdist = n.getDist();
            if (cdist >= MAX_PATH_WEIGHT) break;
            if (seen.containsKey(cnode)) continue;
            seen.put(cnode, true);
```

####Step 3####
-  取得該node連接的所有link **每條link都會存放兩次，src跟destnation會相反**
-  根據 `isDstRooted`，每條link都只取src or dest (因為每條link會出現兩次，所以switch一定不會漏掉)
-  檢查該node是否已經看過了
-  取得該該的cost
-  計算到該neighbor的cost = 本來node的cost + link的cost
-  如果cost比以前學過得更低，那我們就採用這個新的路徑
	- 更新最新的cost資料
  - 更新`nexthoplinks`的資料，記錄到此node所需要的link是哪條
  - 然後把該node重新加入到queue裡面

``` java
            for (Link link: c.links.get(cnode)) {
                Long neighbor;

                if (isDstRooted == true) neighbor = link.getSrc();
                else neighbor = link.getDst();
                // links directed toward cnode will result in this condition
                if (neighbor.equals(cnode)) continue;

                if (seen.containsKey(neighbor)) continue;

                if (linkCost == null || linkCost.get(link)==null) w = 1;
                else w = linkCost.get(link);

                int ndist = cdist + w; // the weight of the link, always 1 in current version of floodlight.
                if (ndist < cost.get(neighbor)) {
                    cost.put(neighbor, ndist);
                    nexthoplinks.put(neighbor, link);
                    log.info("neibhbor = {}",neighbor.toString());
                    //nexthopnodes.put(neighbor, cnode);
                    NodeDist ndTemp = new NodeDist(neighbor, ndist);
                    // Remove an object that's already in there.
                    // Note that the comparison is based on only the node id,
                    // and not node id and distance.
                    nodeq.remove(ndTemp);
                    // add the current object to the queue.
                }
            }
        }
```
####Step 4####
- 利用`nexthoplinks`去創見一個broadcast tree.並且把該tree回傳做為該node的shortest path tree.

``` java
        BroadcastTree ret = new BroadcastTree(nexthoplinks, cost);
        return ret;
    }

```
