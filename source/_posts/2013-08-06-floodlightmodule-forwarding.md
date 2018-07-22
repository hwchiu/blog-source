---
layout: post
title: 'FloodlightModule-Forwarding'
date: 2013-08-06 15:26
comments: true
tags:
	- Floodlight
	- SDN
	- Openflow
	- Network
	- SourceCode
keywords: SDN,Controller,Floodlight
---
Floodlight中，最基本用來轉送封包的module就是Forwarding Module,這邊稍為介紹一下心得:


##Architecture##
Forwarding 擴充了 *ForwardingBase*,
ForwardingBase位於/routing/底下，之後會再詳細介紹這個module.
其主要功能就是幫忙把一個route給送到對應的switches，透過flow-modify的封包來寫入flow-entry到每個路徑上的switch。
當有封包近來的時候就會呼叫`processPacketInMessage`此function
ForwardingBase本身並沒有實作該function，把這判斷的部分交給其他的module處理，這邊就是由forwarding modules來處理。
##Hight-Level Overview##
當有一個PacketIn event送到controller時，Forwarding中會根據ㄧ些已經決定的decision (如firewall)來決定如何處理
如果沒有決定的話，就採用預設的行為處理

<!--more-->


1. Drop
2. Flood (default for broadcast or multicast in Ethernet header)
3. Forward (default)

##Low-Level implementation##
###Drop###

- Create Openflow Flow-Modify Packet with no action ( no action means drop)
- Send Flow-Modify Packet to switch.



**得到一個Openflow Flow-Modify類型的封包**
`OFFlowMod fm =(OFFlowMod) floodlightProvider.getOFMessageFactory().getMessage(OFType.FLOW_MOD);`

**設定一個Actions,然後不增加任何action,這樣就會事drop的行為**
`List<OFAction> actions = new ArrayList<OFAction>();`


**設定Flow-Modify Packet的ㄧ些欄位，譬如HardTimeout,IdleTimeout...,這邊沒有設定Command預設就是flow_add**
``` java
fm.setCookie(cookie)
  .setHardTimeout((short) 0)
  .setIdleTimeout((short) 5)
  .setBufferId(OFPacketOut.BUFFER_ID_NONE) 
  .setMatch(match)
  .setActions(actions)
  .setLengthU(OFFlowMod.MINIMUM_LENGTH); 
```

**把訊息藉由messageDamper送給switch**
`messageDamper.write(sw, fm, cntx);`

###Flood###

- Check the ingress port is allowed broadcast ( according broadcast tree)
- Create Packout packet with Flood action
- Send Packout to switch.


**根據BroadCast Tree判斷發送PacketIn Event的{swtich,port}是否能夠廣播，避免造成broadcast storm**
````
if (topology.isIncomingBroadcastAllowed(sw.getId(),
    pi.getInPort()) == false) {
    return;
}
```
**創造一個Packet Out的封包**
`OFPacketOut po =(OFPacketOut) floodlightProvider.getOFMessageFactory().getMessage(OFType.PACKET_OUT)`

**創造actions,放入一個flood的action,根據ㄧ些property來決定要送到哪個logical port**
```
List<OFAction> actions = new ArrayList<OFAction>();
if (sw.hasAttribute(IOFSwitch.PROP_SUPPORTS_OFPP_FLOOD)) {
    actions.add(new OFActionOutput(OFPort.OFPP_FLOOD.getValue(),
    (short)0xFFFF));
} else {
    actions.add(new OFActionOutput(OFPort.OFPP_ALL.getValue(),
    (short)0xFFFF));
}
po.setActions(actions);
po.setActionsLength((short) OFActionOutput.MINIMUM_LENGTH);
```

**把封包的資料一併傳下去，然後flood**
**如果PacketIn是送bufferID而不是packetData的話，這邊是否要額外判斷???**

```

byte[] packetData = pi.getPacketData();
poLength += packetData.length;
po.setPacketData(packetData);
```
**把訊息藉由messageDamper送給switch**
`messageDamper.write(sw, po, cntx);`


###Forward###

- check we know the desination device
- check source device and destination device are same cluseter
- find all attach switch
- find route between souce device and destination device
- use *forwardingBase*'s method to push a route to all swith which on route.


**先取得source 跟 destination device**
**每個device 是用IP、MAC、VLAN來做為區別的**
`IDevice dstDevice = IDeviceService.fcStore.get(cntx, IDeviceService.CONTEXT_DST_DEVICE);`
`IDevice srcDevice = IDeviceService.fcStore.get(cntx, IDeviceService.CONTEXT_SRC_DEVICE);`

**接下來根據pkacetIN進來的switch取得其所屬的cluster.**
`Long srcIsland = topology.getL2DomainId(sw.getId());`

**去探訪destination device所連接到的switch,看看是否有跟發生PacketIn的switch是在同一個Cluster,
是的話才有辦法轉送，否則就Flood出去**

```
for (SwitchPort dstDap : dstDevice.getAttachmentPoints()) {
    Long dstSwDpid = dstDap.getSwitchDPID();
    Long dstIsland = topology.getL2DomainId(dstSwDpid);`

if ((dstIsland != null) && dstIsland.equals(srcIsland)) 
on_same_island = true;
```


**取得source / destination device所連接到的所有switch**
**目前還不是很清楚怎樣的情形下，可以一個device連接到多個switch 也許用hub吧**

`SwitchPort[] srcDaps = srcDevice.getAttachmentPoints();
`SwitchPort[] dstDaps = dstDevice.getAttachmentPoints();`


**利用routingEngine來取得兩個switch間的最短路徑 (dijstra)**
`Route route = routingEngine.getRoute(srcDap.getSwitchDPID(),(short)srcDap.getPort(),dstDap.getSwitchDPID(),
(short)dstDap.getPort(), 0);`

**接者透過ForwardingBase的pushRoute,會把路徑上所有的switch都發送一個Flow-Modify的封包**
`pushRoute(route, match, wildcard_hints, pi, sw.getId(), cookie,cntx, requestFlowRemovedNotifn,
						false, OFFlowMod.OFPFC_ADD);`


**處理完這組switch後，繼續嘗試其他連接的switch**
`iSrcDaps++;`
`iDstDaps++;`

##結論##
Forwarding是個很基本的module,原始的情況下就是把封包給forward或是flood的而已，
目前裡面的設計是希望能夠取得多個attach points,但是我目前嘗試各種拓樸，都沒有辦法讓一個device連接到多個switch,不知道是否要使用hub之類的東西來完成，這部分要再嘗試看看。



