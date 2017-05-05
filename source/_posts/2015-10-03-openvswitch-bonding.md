---
layout: post
title: 'OpenvSwitch Bonding'
date: 2015-10-03 13:32
comments: true
categories: 
---
Introduction
------------
Openvswitch 目前 (2.3.1) 總共支援 3種bonding mode，分別是
- Active-backup
- balance-slb
- balance-tcp

<!--more-->

Active-backup
-------------
這種 mode 的用途主要在於穩定，平常只會使用 bonding 中的其中一條 link 進行傳輸，當該 link down 時，會馬上切換到其他 link 繼續傳輸。本質上沒有辦法提升 throughput。

Balance-slb
-----------
這種 mode 的 hash 方式是根據封包的 source MAC + vlan tag來處理，可以參考此篇[文章](http://openvswitch.org/pipermail/dev/2011-July/010028.html)有更詳細的說明

Balance-tcp
-----------
這種 mode 的 hash 是根據封包的 L2/L3/L4 header 來處理的，所以每條 connection 可能會走不同的 link 出去，但是相同 connection 則會一直固定以避免發生 out of order 之類的事情。 

註: 如果使用 linux 本身的 round-robin bonding 則可以讓一條 connction 走不同的 link，兩條 1G 的link大概可以衝到 1.5G左右


對於 Balance-slb 以及 Balance-tcp 來說，這邊還能夠再增加是否要開啟 LACP (802.3ad) 的設定。
當開啟 LACP 後，會使用 balance-slb 或是 balance-tcp 的 hash method 當作其分配封包的方式。
唯一要注意的是 balance-tcp 一定要搭配 LACP 才可以使用。

Commands
--------
### 創造 bonding
- ovs-vsctl add-br my_test
- ovs-vsctl add-bond my_test bond0 eth0 eth1 eth2 

此指令會在 my_test 此 bridge 上面創造一個 bonding interface *bond0*，此 bonding interface 會將 eth0, eth1, eth2 給綁起來

### 改變 bonding mode
預設的 bonding mode 是 active-back，可以再創造的時候設定或是之後再改變
- ovs-vsctl add-bond my_test bond0 eth0 eth1 eth2 bond_mode=balance-slb
- ovs-vsctl set port my_test bond_mode=balance-slb

### 看 bonding 相關資訊
- ovs-appctl bond/show bond0
- ovs-appctl bond/list bond0
- ovs-appctl bond/hash bond0 (可以看 hash 對應的 slave interface)
- ovs-appctl bond/migrate (能夠將某 hash 從某slave 搬移到別的slave)

Testing
-------
###測試配備如下
- HP ProCurve Switch 2824 (J4903A)
	- 針對 LACP 的實驗，必須要在這邊開啟 LACP
- Linux PC *1
- Windows PC *2

###測試拓樸一
<img class="left" src="http://i.imgur.com/RbsM1rF.png">

- linux PC 上面安裝 OpenvSwitch，並且與 HP Switch 以兩個 1G 的 port 進行 bonding。
- 兩台 Windows PC 都連接在 HP Switch
- Linux PC 與 Windows PC 以 iperf 作為產生流量的工具
- TX 測試
	-	linux PC 跑 iperf client (-P4) 分別打到兩台 windows PC
  - windows PC 分別跑 iperf server
- RX 測試
	-	linux PC 跑 iperf server 
  - windows PC 分別跑 iperf client，分別用 iperf -P4 去連接 linux PC
- 實驗數據 (TX、RX是分開跑)
![](http://i.imgur.com/nDNFI3L.png)
  - 數據分析方式請看最後面
- 分析
  - Active-backup 就只有用一條link傳輸，沒有辦法達到 speed up 的效果
  - Balance-slb without lacp 因為我們的 source mac 都是 linuxPC 本身，所以也只會用一條 link 來傳輸，本身沒有任何幫助
  - Balance-slb with lacp 因為有打開 LACP 的功能，所以從 switch 回來的封包會分兩個 link 去送，所以 RX 可以看到有明顯的上升，大概1.9G左右
  - Balance-tcp with lacp 因為是根據 L2/L3/L4 來進行 hash，所以同一個 Host 發出的不同 connection 可以分散在不同 link上，所以 TX 的速度也有明顯上升
                                           


###測試拓樸二
<img class="left" src="http://i.imgur.com/Fm3Coea.png">
- linux PC 上面安裝 OpenvSwitch，並且與 HP Switch 以兩個 1G 的 port 進行 bonding。
- linux PC 上面設定兩個獨立的 network namespace，並且把此兩個 NS 的給掛到 OpenvSwitch 上面
- 兩台 Windows PC 都連接在 HP Switch
- Linux PC 上的 NS 與 Windows PC 以 iperf 作為產生流量的工具
- TX 測試
	-	NS 分別跑 iperf client (-P4)
  - windows PC 分別跑 iperf server
  - 一個 NS 對應一個 Windows PC
- RX 測試
	-	windows PC 分別跑 iperf client (-P4)
  - NS 分別跑 iperf server
  - 一個 NS 對應一個 Windows PC  
- 實驗數據 (TX、RX是分開跑)
 - 數據分析方式請看最後面
![](http://i.imgur.com/Scvj1Hp.png)
- 分析
  - Balance-slb without lacp 因為我們的 source mac 是兩台不同的 NS ，所以有機會兩台 NS 的 MAC會被 hash 到不同的 link，所以 TX 的速度也有明顯上升
  - Balance-slb with lacp TX 方面理由如上， RX是因為 LACP 而加速
  - Balance-tcp with lacp 理由如同實驗一


###個人心得
- 只有一台 host 本身的話，其實跑 OVS 的bonding沒有太大效果，除非你外面有支援 LACP的switch 可以用，不然就直接用 linux 原先的 XOR 之類的hash就好
- 若是在多 VM 的環境下，這時候有 balance-slb 與 balance-tcp 可以考慮(假設想要 speed up)，這兩個主要考慮的點在於
使用 balance-slb 的話，會讓同一個 VM 的所有流量都走同一個 interface 出去，所以若當前其他 VM 都閒置的情況下，該 VM 還是只能用到一條 link 的資源。
若採用 balance-tcp 的話，則會依照 connction 來分，所以不論何種情況都能夠盡量使用每條 link 的資源


###其他
- 流量的觀察方式是在 linux PC 上面透過 /proc/net/dev 週期性觀察兩張 slave interface 的 TX/RX counter計算得知，週期性為一秒。


