---
title: Floodlight Core RestAPI - part1
keywords: 'Floodlight, API, Restful'
date: '2013-11-24 05:58'
comments: true
tags:
  - SDN
  - Floodlight
  - Java
  - Network
  - SourceCode
abbrlink: 51608
description: 本文基於 SDN Controller Floodlight 的原始碼進行了一次簡單的分析，藉由分析這些原始碼更可以瞭解每個開放出來的 Restful API 該怎麼使用。相對於文件的更新速度，程式碼本身的迭代速度更為敏捷，因此常常會發生文件跟不上實際運行功能的案例。藉由學習閱讀原始碼，我們可以更快也更清楚的掌握當前這些開源軟體的發展狀態，甚至也能夠貢獻社群幫忙補齊文件。

---

# Preface

Floodlight Openflow Controller 預設就有Rest Server 並且提供對應的Rest API供使用者呼叫使用

再core module這邊，目前提供了8種restAPI使用

- /wm/core/switch/all/$statType/json
- /wm/core/swtich/$switchi>/$statType$/json
- /wm/core/controller/switches/json
- /wm/core/role/json
- /wm/core/counter/$counterTitle/json
- /wm/core/counter/$switchId/$counterName$/json
- /wm/core/memory/json
- /wm/core/module/{all}/json

第一篇主要講前面兩個，關於switch information方面。


# Usage
這兩個RestAPI做的事情就是對switch 發送 Openflow status request的封包去詢問相關的訊息
發送的種類就是 **statType**。以下是目前的類型

- port
- queue
- flow
- aggregate
- desc
- table
- features
- host

兩個API的差別只有再於一個是針對所有的swtich去發送請求，另一個是針對特定的switch去請求。




# Implement

1. `core/web/CoreWebRoutable.java` 裡面可以發現core像IRestApiService註冊了下列事件
```
router.attach("/switch/all/{statType}/json", AllSwitchStatisticsResource.class);
router.attach("/switch/{switchId}/{statType}/json", SwitchStatisticsResource.class);
```
這邊可以看到，
當使用者透過  `/wm/core/switch/all/{statType}/json`時最後會透過`AllSwitchStatisticsResource`去處理
如果透過的指定swtich id的方式，則會透過 `SwitchStatisticsResource`這個物件來處理。

## 特定switch id

2. `core/web/SwitchStatisticsResource.java` 可以看到下列的程式碼
``` java
    @Get("json")
    public Map<String, Object> retrieve() {
        HashMap<String,Object> result = new HashMap<String,Object>();
        Object values = null;

        String switchId = (String) getRequestAttributes().get("switchId");
        String statType = (String) getRequestAttributes().get("statType");

        if (statType.equals("port")) {
            values = getSwitchStatistics(switchId, OFStatisticsType.PORT);
        } else if (statType.equals("queue")) {
            values = getSwitchStatistics(switchId, OFStatisticsType.QUEUE);
        } else if (statType.equals("flow")) {
            values = getSwitchStatistics(switchId, OFStatisticsType.FLOW);
        } else if (statType.equals("aggregate")) {
            values = getSwitchStatistics(switchId, OFStatisticsType.AGGREGATE);
        } else if (statType.equals("desc")) {
            values = getSwitchStatistics(switchId, OFStatisticsType.DESC);
        } else if (statType.equals("table")) {
            values = getSwitchStatistics(switchId, OFStatisticsType.TABLE);
        } else if (statType.equals("features")) {
            values = getSwitchFeaturesReply(switchId);
        }

        result.put(switchId, values);
        return result;
    }
```

這邊可以看到`retrieve`會先取得使用者輸入的swtichId以及對應的statType.
接者透過 `getSwitchStatistics` 這個function去取得資料
最後透過 `result.put(switchId, values)` 把該資料跟該dpid綁在一起，方便JSON的格式回傳

3. `core/web/SwitchResourceBase.java` 中可以看到關於 `getSwitchStatistics`的定義


``` java
    protected List<OFStatistics> getSwitchStatistics(long switchId,
                                                     OFStatisticsType statType) {
        IFloodlightProviderService floodlightProvider =
                (IFloodlightProviderService)getContext().getAttributes().
                    get(IFloodlightProviderService.class.getCanonicalName());

        IOFSwitch sw = floodlightProvider.getSwitch(switchId);
        Future<List<OFStatistics>> future;
        List<OFStatistics> values = null;
        if (sw != null) {
            OFStatisticsRequest req = new OFStatisticsRequest();
            req.setStatisticType(statType);
            int requestLength = req.getLengthU();
```
- 先透過floodlightProvider 取得該`switchId`所對應IOFSitch object.
- 初始化查詢結果以及需要的容器 `future`, `values`
- 如果該dpid對應的switch存在，則先產生一個`OFStatisticsRequest`的物件，等等就要透過這個物件去發送請求。

``` java
            if (statType == OFStatisticsType.FLOW) {
                OFFlowStatisticsRequest specificReq = new OFFlowStatisticsRequest();
                OFMatch match = new OFMatch();
                match.setWildcards(0xffffffff);
                specificReq.setMatch(match);
                specificReq.setOutPort(OFPort.OFPP_NONE.getValue());
                specificReq.setTableId((byte) 0xff);
                req.setStatistics(Collections.singletonList((OFStatistics)specificReq));
                requestLength += specificReq.getLength();
            } else if (statType == OFStatisticsType.AGGREGATE) {
                OFAggregateStatisticsRequest specificReq = new OFAggregateStatisticsRequest();
                OFMatch match = new OFMatch();
                match.setWildcards(0xffffffff);
                specificReq.setMatch(match);
                specificReq.setOutPort(OFPort.OFPP_NONE.getValue());
                specificReq.setTableId((byte) 0xff);
                req.setStatistics(Collections.singletonList((OFStatistics)specificReq));
                requestLength += specificReq.getLength();
            } else if (statType == OFStatisticsType.PORT) {
                OFPortStatisticsRequest specificReq = new OFPortStatisticsRequest();
                specificReq.setPortNumber(OFPort.OFPP_NONE.getValue());
                req.setStatistics(Collections.singletonList((OFStatistics)specificReq));
                requestLength += specificReq.getLength();
            } else if (statType == OFStatisticsType.QUEUE) {
                OFQueueStatisticsRequest specificReq = new OFQueueStatisticsRequest();
                specificReq.setPortNumber(OFPort.OFPP_ALL.getValue());
                // LOOK! openflowj does not define OFPQ_ALL! pulled this from openflow.h
                // note that I haven't seen this work yet though...
                specificReq.setQueueId(0xffffffff);
                req.setStatistics(Collections.singletonList((OFStatistics)specificReq));
                requestLength += specificReq.getLength();
            } else if (statType == OFStatisticsType.DESC ||
                       statType == OFStatisticsType.TABLE) {
                // pass - nothing todo besides set the type above
            }
            req.setLengthU(requestLength);
```
 - 這邊就是針對type的請求，使用不同格式的封包。
 - 以flow為例子， flow request 會使用`OFMatch`去尋找所有match的flow,有mathc的flow才會回傳狀態資訊

 	* `setWildcards(0xffffffff)`: 這樣就能夠match 所有的flow
	* `specificReq.setOutPort(OFPort.OFPP_NONE.getValue())`: 這邊使用OFPP_NONE就是代表在match flow的時候，不會去看該flow entry的output port.
  * `specificReq.setTableId((byte) 0xff)`: 把tableId設定成0xff就是代表對所有的table都去詢問。
  * 最後設定一些相關資訊，並且更新整個request的長度
  * `req.setLengthU(requestLength)`設定整個request packet的最後長度

``` java
            try {
                future = sw.queryStatistics(req);
                values = future.get(10, TimeUnit.SECONDS);
            } catch (Exception e) {
                log.error("Failure retrieving statistics from switch " + sw, e);
            }
        }
        return values;
    }
```

-  接下來把該 request的封包送給switch,然後使用一個`future`的物件來取得回傳結果,透過future去發送一個非同步的要求，如果10秒內沒有辦法把該任務給完成，就會發出例外直接停止。
- 最後把結果給回傳回去。


## 所有switch

- `core/web/AllSwitchStatisticsResource.java` 中可以觀察倒整個code

``` java
    @Get("json")
    public Map<String, Object> retrieve() {
        String statType = (String) getRequestAttributes().get("statType");
        return retrieveInternal(statType);
    }
```

- 這邊可以看到，會先從statType中取得使用者要求的type,接者再呼叫`retrieveInternal()`來取得結果並回傳給使用者

``` java
    public Map<String, Object> retrieveInternal(String statType) {
        HashMap<String, Object> model = new HashMap<String, Object>();

        OFStatisticsType type = null;
        REQUESTTYPE rType = null;

        if (statType.equals("port")) {
            type = OFStatisticsType.PORT;
            rType = REQUESTTYPE.OFSTATS;
        } else if (statType.equals("queue")) {
            type = OFStatisticsType.QUEUE;
            rType = REQUESTTYPE.OFSTATS;
        } else if (statType.equals("flow")) {
            type = OFStatisticsType.FLOW;
            rType = REQUESTTYPE.OFSTATS;
        } else if (statType.equals("aggregate")) {
            type = OFStatisticsType.AGGREGATE;
            rType = REQUESTTYPE.OFSTATS;
        } else if (statType.equals("desc")) {
            type = OFStatisticsType.DESC;
            rType = REQUESTTYPE.OFSTATS;
        } else if (statType.equals("table")) {
            type = OFStatisticsType.TABLE;
            rType = REQUESTTYPE.OFSTATS;
        } else if (statType.equals("features")) {
            rType = REQUESTTYPE.OFFEATURES;
        } else {
            return model;
        }

```

- 根據使用者的type, 設定 `type`跟 `rType`兩種變數， 其中rType是用來做`features request`的。

``` java
        IFloodlightProviderService floodlightProvider =
                (IFloodlightProviderService)getContext().getAttributes().
                    get(IFloodlightProviderService.class.getCanonicalName());
        Set<Long> switchDpids = floodlightProvider.getAllSwitchDpids();
        List<GetConcurrentStatsThread> activeThreads = new ArrayList<GetConcurrentStatsThread>(switchDpids.size());
        List<GetConcurrentStatsThread> pendingRemovalThreads = new ArrayList<GetConcurrentStatsThread>();
        GetConcurrentStatsThread t;
        for (Long l : switchDpids) {
            t = new GetConcurrentStatsThread(l, rType, type);
            activeThreads.add(t);
            t.start();
        }

```
- 這邊會先透過`floodlightProvider` 取得所有的swtich dpid
- 然後跑一個for迴圈，針對每個switch都去發送一個request,這邊採用了thread的方式來發送，並且把這個thread給記錄下來。

``` java
        for (int iSleepCycles = 0; iSleepCycles < 12; iSleepCycles++) {
            for (GetConcurrentStatsThread curThread : activeThreads) {
                if (curThread.getState() == State.TERMINATED) {
                    if (rType == REQUESTTYPE.OFSTATS) {
                        model.put(HexString.toHexString(curThread.getSwitchId()), curThread.getStatisticsReply());
                    } else if (rType == REQUESTTYPE.OFFEATURES) {
                        model.put(HexString.toHexString(curThread.getSwitchId()), curThread.getFeaturesReply());
                    }
                    pendingRemovalThreads.add(curThread);
                }
            }

            // remove the threads that have completed the queries to the switches
            for (GetConcurrentStatsThread curThread : pendingRemovalThreads) {
                activeThreads.remove(curThread);
            }
            // clear the list so we don't try to double remove them
            pendingRemovalThreads.clear();

            // if we are done finish early so we don't always get the worst case
            if (activeThreads.isEmpty()) {
                break;
            }

            // sleep for 1 s here
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                log.error("Interrupted while waiting for statistics", e);
            }
        }

        return model;
    }
```
- 這邊是用來收集所有thread的結果，並且統合後把結果回傳的地方
- 最外圈是一個12秒的迴圈，然後裡面會針對之前記錄的所有thread去跑，如果該thread任務已經結束，狀態是`TERMINATED`
就會把該thread記錄倒一個 **記錄要被移除thread**pendingRemovalThreads,然後從該thread取回請求結果，並記錄下來。
- 從pendingRemovalThreads拿出所有thread，並且把對應於activeThread中的那份給刪除掉
- 如果最後`activeThread`已經是空的，就代表所有結果都會來了，提前結束。


``` java
    protected class GetConcurrentStatsThread extends Thread {
...
        @Override
        public void run() {
            if ((requestType == REQUESTTYPE.OFSTATS) && (statType != null)) {
                switchReply = getSwitchStatistics(switchId, statType);
            } else if (requestType == REQUESTTYPE.OFFEATURES) {
                featuresReply = getSwitchFeaturesReply(switchId);
            }
        }
...
}
- 這個thread 會根據前面傳進來的type,去呼叫`SwitchResourceBase.java`中的`getSwitchStatistics`來取得結果。
```
