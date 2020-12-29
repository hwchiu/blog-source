---
layout: post
title: Mininet with different network subnet (v2)
date: '2014-09-19 13:00'
comments: true
tags:
  - SDN
  - Openflow
  - Mininet
  - Network
keywords: 'SDN,Mininet,Network'
abbrlink: 61927
---
上一篇[mininet-and-network-subnet](http://logdown.com/account/posts/203260-mininet-and-network-subnet/edit)中提到如何在mininet中創造不同subnet的網路，並且透過手動下flow的方式讓不同subnet的hosts可以互相溝通。
而本篇的重點在於提供另外一種方式來創造不同subnet的網路，讓研究者不需要再手動一直輸入**ifconfig**,**route add ...**等指令，能夠更簡潔的去創造不同subtnet的網路。

<!--more-->

Solution
--------
在本篇中，我們直接撰寫**mininet**的python script來模擬網路，基本的撰寫教學請參考mininet官方文件就有了。本篇主要是針對不同subnet的host要如何創建。
首先，在我們創造hosts的時候，可以透過**ip**這個參數來控制此host的預設ip位置，這時候我們就可以設定**10.0.0.0/24**或是**20.0.0.0/24**等ip給予欲創建的host，這樣就可以省掉之前的**ifconfig**的步驟。
接下來，我們要處理Default gateway的問題，這邊也有**defaultRoute**的參數可以使用，這邊我們就可以輸入**defaultRoute='h1-eth0'**來處理，這樣就可以省掉之前所輸入的**route add default gw**的步驟。
這兩個參數都正確填寫完畢後，我們就創立好了不同subnet的網路，並且基本的設定已經完成了，接下來就按照上一篇的說明來將flow entry給寫入switch即可。

###完整的mininet python script
``` python
#!/usr/bin/python

from mininet.net import Mininet
from mininet.node import Controller, RemoteController, OVSController
from mininet.node import CPULimitedHost, Host, Node
from mininet.node import OVSKernelSwitch, UserSwitch
from mininet.node import IVSSwitch
from mininet.cli import CLI
from mininet.log import setLogLevel, info
from mininet.link import TCLink, Intf

def myNetwork():

    net = Mininet( topo=None,
                   build=False,
                   ipBase='10.0.0.0/8')

    info( '*** Adding controller\n' )
    c0=net.addController(name='c0',
                      controller=RemoteController,
                      ip='127.0.0.1')
    info( '*** Add switches\n')
    s1 = net.addSwitch('s1', cls=OVSKernelSwitch)
    info( '*** Add hosts\n')

    h1 = net.addHost('h1', cls=Host, mac='00:00:00:00:00:01', ip='10.0.0.1/24', defaultRoute='h1-eth0')
    h2 = net.addHost('h2', cls=Host, mac='00:00:00:00:00:02', ip='20.0.0.1/24', defaultRoute='h2-eth0')
    h3 = net.addHost('h3', cls=Host, mac='00:00:00:00:00:03', ip='30.0.0.1/24', defaultRoute='h3-eth0')

    info( '*** Add links\n')
    linkBW = {'bw':100}
    net.addLink(h1, s1, cls=TCLink , **linkBW)
    net.addLink(h2, s1, cls=TCLink , **linkBW)
    net.addLink(h3, s1, cls=TCLink , **linkBW)
    info( '*** Starting network\n')
    net.build()
    info( '*** Starting controllers\n')
    for controller in net.controllers:
        controller.start()
    info( '*** Starting switches\n')
    net.get('s1').start([c0])
    info( '*** Configuring switches\n')

    CLI(net)
    net.stop()

if __name__ == '__main__':
    setLogLevel( 'info' )
    myNetwork()
```


###測試用的flow entries

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
