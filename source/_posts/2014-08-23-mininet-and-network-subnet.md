---
layout: post
title: 'Mininet with different network subnet'
date: 2014-08-23 14:17
comments: true
categories: 
---
Introduction
------------
我們使用 `mn --topo single,3 --mac`創造一個有不同subnet的拓墣，使用`--mac`使得所有host的**MAC Address**更加簡單，能夠使得此實驗變得容易。
我們目標是要讓這三個不同subnet的host都能夠互相溝通。

<!--more-->
網路架構如下圖
![topology.png](http://user-image.logdown.io/user/415/blog/415/post/203260/DaB0okyvRsG7mvB0jt0j_topology.png)

在此實驗中，我們並不採用任何controller來控制所有封包，單純就手動下**flow entry**來處理所有的封包，一旦了解了這中間的道理，要自己撰寫ＡＰＰ處裏此情況就不會太難了。

Solutions
---------
首先，mininet創造出來網路後，預設會讓所有的host都屬於相同的**network subnet** 10.0.0.0/24，因此在實驗開始前，我們要先修改其餘host的設定，改變其network subnet。
-	在**mininet**的環境中執行下列指令
- **h2 ifconfig h2-eth0 20.0.0.1**
- **h3 ifconfig h3-eth0 30.0.0.1**

接下來，我們先執行**h1 ping h3**，這時候我們會看到有錯誤訊息 **connect: Network is unreachable**。這個原因是因為對於host1來說，host2是不一樣的**network subnet**，此時會將該封包轉送到本身subnet的**gateway**來處理，但是該host不知道**gateway**在哪裡，因此我們要幫他們加上**route for default gateway**。
-	在**mininet**的環境中執行下列指令
- **h1 route add default gw 10.0.0.254 h1-eth0**
- **h2 route add default gw 20.0.0.254 h2-eth0**
- **h3 route add default gw 30.0.0.254 h3-eth0**

接下來，我們繼續執行`h1 ping h3`，此時會得到下列的訊息

```
mininet> h1 ping h3
PING 30.0.0.1 (30.0.0.1) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
From 10.0.0.1 icmp_seq=2 Destination Host Unreachable
From 10.0.0.1 icmp_seq=3 Destination Host Unreachable
From 10.0.0.1 icmp_seq=4 Destination Host Unreachable
```
到這步驟後，因為我們還沒有寫入任何的**flow entry**，所以網路不通是正常的。在處理**ICMP 封包**前，我們必須要先處理**ARP**的封包。
這邊我們先在**mininet**那邊持續的執行**h1 ping h3**。同時，我們開啟第二個視窗，執行**tcpdump -vvv -i s1-eth1**，我們會得到下列的訊息
```sh
tcpdump: WARNING: s1-eth1: no IPv4 address assigned
tcpdump: listening on s1-eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
20:07:04.639862 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.0.0.254 tell 10.0.0.1, length 28
20:07:05.639859 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.0.0.254 tell 10.0.0.1, length 28
20:07:06.639895 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.0.0.254 tell 10.0.0.1, length 28
20:07:07.639856 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.0.0.254 tell 10.0.0.1, length 28
```
由這邊可以發現，Host 1透過**arp**在詢問其**gateway**相關資訊，但是麻煩的是，在此網路中，我們並沒有真的一個Device的ip是該**gateway**，為了解決這個問題，我們有兩個選擇
1. 弄一個Host出來，當作gateway去處理
2. 弄個**arp proxy**來處理，這部分在**OpenDayLight**中預設有提供此module，讓controller假裝自己是**gateway**來處理此問題。

由於本實驗並沒有採用任何controller，因此我們要手動修改switch，讓她覺得自己是**gateway**，能夠回**arp reply**給Host。
-	在**mininet**的環境中執行下列指令
- **s1 ifconfig s1:0 10.0.0.254**
- **s1 ifconfig s1:1 20.0.0.254**
- **s1 ifconfig s1:2 30.0.0.254**

我們令**s1**這個**interface**擁有三個ip，這些ip都代表每個**network subnet**的**gateway**ip，接下來為了讓switch自己幫我們處理所有**arp request for gateway**，我們加入下列**flow entry**到s1中

-	在**mininet**的環境中執行下列指令
- **sh ovs-ofctl add-flow s1 "table=0,priority=65535,arp,arp_tpa=10.0.0.254 actions=LOCAL"**
- **sh ovs-ofctl add-flow s1 "table=0,priority=65535,arp,arp_tpa=20.0.0.254 actions=LOCAL"**
- **sh ovs-ofctl add-flow s1 "table=0,priority=65535,arp,arp_tpa=30.0.0.254 actions=LOCAL"**

上面這三個**flow entry**會把所有**arp request for gateway**的封包都導入本地的OS去處理，因此這些封包就會進入到
**s1:0,s1:1,s1:2**去處理，並且回覆一個**arp reply**。這些**arp reply**都會再度的進到ＯＶＳ內，為了處理這些封包，我們要根據他的**destination ip address**把它給送回去對應的Ｈost。

- **sh ovs-ofctl add-flow s1 "table=0,priority=1,arp,nw_dst=10.0.0.1,actions=output:1"**
- **sh ovs-ofctl add-flow s1 "table=0,priority=1,arp,nw_dst=20.0.0.1,actions=output:2"**
- **sh ovs-ofctl add-flow s1 "table=0,priority=1,arp,nw_dst=30.0.0.1,actions=output:3"**

這些完畢後，**arp**封包就能夠正常處理了，接下來為了處理**ICMP**，我們要再做一些設定，在此實驗中，我們同時測試**multiple table**的功用，因此我們決定把**ICMP routing**的部分放到第二個table去處理。
首先，我們先在**table 0**加入一個**flow entry**，把剛剛沒有被**arp**處理掉的封包都送到**table 1**去處理。 
- **sh ovs-ofctl add-flow s1 "table=0,priority=0,actions=resubmit(,1)"**

接者，在**table 1**，因為switch的身份很類似**router**，因此我們要修改所有封包的**destination MAC Address**。
- **sh ovs-ofctl add-flow s1 "table=1,icmp,nw_dst=10.0.0.1,actions=mod_dl_dst=00:00:00:00:00:01,output:1"**
- **sh ovs-ofctl add-flow s1 "table=1,icmp,nw_dst=20.0.0.1,actions=mod_dl_dst=00:00:00:00:00:02,output:2"**
- **sh ovs-ofctl add-flow s1 "table=1,icmp,nw_dst=30.0.0.1,actions=mod_dl_dst=00:00:00:00:00:03,output:3"**

最後執行`h1 ping h3`，就會順利的通了，以下整理一下**flow table**中的所有**flow entry**
```
#Those two flow will handle the arp-request for the gateway, it will send the arp-request to s1
table=0,priority=65535,arp,arp_tpa=10.0.0.254 actions=LOCAL
table=0,priority=65535,arp,arp_tpa=20.0.0.254 actions=LOCAL
table=0,priority=65535,arp,arp_tpa=30.0.0.254 actions=LOCAL
table=0,priority=1,arp,nw_dst=10.0.0.1,actions=output:1
table=0,priority=1,arp,nw_dst=20.0.0.1,actions=output:2
table=0,priority=1,arp,nw_dst=30.0.0.1,actions=output:3
table=0,priority=0,actions=resubmit(,1)

#table1  - forward/route
table=1,icmp,nw_dst=10.0.0.1,actions=mod_dl_dst=00:00:00:00:00:01,output:1
table=1,icmp,nw_dst=20.0.0.1,actions=mod_dl_dst=00:00:00:00:00:02,output:2
table=1,icmp,nw_dst=30.0.0.1,actions=mod_dl_dst=00:00:00:00:00:03,output:3
```







