---
title: Nox-Spanning_Tree
keywords: 'NOX,SDN,Topology,Tree'
date: '2013-09-02 07:01'
comments: true
tags:
  - SDN
  - Openflow
  - Network
  - Nox
  - Python
abbrlink: 37403
description: 對於 SDN Controller 來說，最基本的功能就是要可以傳輸封包，然而在這種集中式管理的情況下，傳統的 Spanning Tree Protocol 不會運行。因此 Controller 本身要有辦法判斷當前的網路拓墣中是否有迴圈以避免產生廣播風暴。本文會透過觀察原始碼的方式來研究在 NOX Conroller 是如何實現的。

---

# Preface
Spanning_tree 是nox的一個module.
nox會藉由此moudle來維護spanning tree,避免封包在廣播的時候會產生broadcast storm.

# Introduction
## __init__
這邊做的是一些成員的初始化，包括一些set的初始。

## install
首先會呼叫```update_lldp_send_period```更新一次lldp發送的頻率

會根據port的數量去決定LLDP發送的頻率,目前FLOO_D_WAIT_TIME 預設是10秒，代表10秒內要把平均的送出LLDP出去。

接者會去註冊一些相關事件
```python=
self.register_for_datapath_join ( self.dp_join )
self.register_for_datapath_leave( self.dp_leave )
self.register_for_port_status( self.handle_port_status )
self.register_for_packet_in( self.handle_packet_in)
```
這段還不是很清楚，但是感覺是讓一些變數在各module之間互通使用的。
`self.bindings = self.resolve(pybindings_storage)`

這段則是透過reactor把命令給延緩1秒後執行
意思就是這行結束一秒後，就會自己執行update_spanning_tree.
`self.post_callback(1, self.update_spanning_tree)`

## dp_join
當有switch 與controller連線之後，便會呼叫此function來做處理。

如果Nox本身不認得該switch的話，就會去紀錄該switch有哪些port
如果該port的port number 小於 OFPP_MAX(65280)的話，就進行相關設定

設定該port的起始時間
預設該port是不能flood的，這樣可以避免新的PORT一出現就會使得spanning tree出問題。
然後發送一個port_modify的封包去把該port給設定成no_flood

最後紀錄該Port，並且重新調整LLDP的值

```python
now = time.time()
ports = {}
for port in stats['ports']:
	ports[port[core.PORT_NO]] = port
	if port[core.PORT_NO] <= openflow.OFPP_MAX:
		port['enable_time'] = now + FLOOD_WAIT_TIME
		port['flood'] = False
		hw_addr = "\0\0" + port[core.HW_ADDR]
		hw_addr = struct.unpack("!q", hw_addr)[0]
		self.ctxt.send_port_mod(dp, port[core.PORT_NO], ethernetaddr(hw_addr),
		openflow.OFPPC_NO_FLOOD, openflow.OFPPC_NO_FLOOD)

self.datapaths[dp] = ports
self.port_count += len(ports)
self.update_lldp_send_period()

```

## dp_leave
當有swtich離開的時候，先檢查該switch是否存在
然後把整體的port_count給調整。

## update_spanning_tree
先利用bindings去取得所有的link，然後把本身的一個callback function傳進去。
接者在FLOOD_PORT_UPDATE_INTERVAL(5 sec)的時間後，呼叫update_spanning_tree.
```python
self.bindings.get_all_links(self.update_spanning_tree_callback)
self.post_callback(FLOOD_PORT_UPDATE_INTERVAL, self.update_spanning_tree)
```

## update_spanning_tree_llback



## handle_port_status


## handle_packet_in


port是flood port或是該封包是LLDP 這兩種情況就直接把該封包傳給下一個module去處理。

除了此情況以外

- 檢查destination的mac address 是否學過
- 檢查destination的ip type = ethernet 且 ip version = ipv4且 ip header = 20byte (5*4)

``` python
if not packet.parsed:
	if packet.type == ethernet.LLDP_TYPE:
		return CONTINUE
```

```
try:
	dst_mac = (struct.unpack('!I', packet.arr[0:4])[0] << 16) + struct.unpack('!H', packet.arr[4:6])[0]
	if dst_mac in self.mac_bypass:
		return CONTINUE

	type = struct.unpack('!H', packet.arr[12:14])[0]
	ipver = struct.unpack('!b', packet.arr[14:15])[0]
	if type == 0x800 and ipver == 0x45:
		dst_ip = struct.unpack('!I', packet.arr[30:34])[0]
			if dst_ip in self.ip_bypass:
				return CONTINUE
	except:
		pass
```

``` python
        try:
            if self.datapaths[dpid][inport]['flood']:
                return CONTINUE
            else:
                log.warn("STOP")
                return STOP
        except KeyError:
            return STOP
```

## update_lldp_send_period

在nox中LLDP的發送情況是要在 FLOOW_WAIT_TIME的時間內 把所有的LLDP都送出去

所以PORT的數量愈多，每個LLDP的間隔就愈短。
預設值

- FLOOW_WAIT_TIME = 10
- MIN_LLDP_SEND_PERIOD = 0.05


``` python
if self.port_count == 0:
	nox.netapps.discovery.discovery.LLDP_SEND_PERIOD = MIN_LLDP_SEND_PERIOD
else:
	nox.netapps.discovery.discovery.LLDP_SEND_PERIOD = min(
		MIN_LLDP_SEND_PERIOD,
		(FLOOD_WAIT_TIME * 1.0) / 2 / self.port_count)
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