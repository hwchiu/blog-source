---
layout: post
title: 'Config Qos on Ovs with Floodlight'
date: 2014-05-05 09:35
comments: true
tags:
	- SDN
	- Openflow
	- Jave
	- Floodlight
	- OpenvSwitch
	- System
keywords: SDN,Controller,Floodlight
---
這邊簡單紀錄一下一種QoS的使用方法，實驗流程如下


Step
----
- Create two different QOS queue on the openvswitch with different limitation.
- Use the **Restful API** to add a flow entry with the action `enqueue`.
- Chagne the flow action when you want to change the QoS behavoir.



Detail
------
- Create topology    
`sudo mn --mac --controller=remote`
- Adding two queues q0(limited 800M) and q1(limited 50M) on s1-eth1 
```
ovs-vsctl -- set port s1-eth1 qos=@newqos -- --id=@newqos create qos type=linux-htb \
queues=0=@q0,1=@q1 -- --id=@q0 create queue other-config:min-rate=200000000 \
other-config:max-rate=800000000 -- --id=@q1 create queue other-config:min-rate=50000 \
other-config:max-rate=50000000
```
- Adding flow entry using q0 and forwarding packet from port2 to port1.
```
curl -d '{"switch": "00:00:00:00:00:00:00:01", "name":"flow-mod-1", "cookie":"0", "priority":"32768","ingress-port":"2","active":"true", "actions":"enqueue=1:0"}' http://127.0.0.1:8080/wm/staticflowentrypusher/json
```
- Type "iperf h2 h1" in mininet   
```  
mininet> iperf h2 h1  
  *** Iperf: testing TCP bandwidth between h2 and h1  
  *** Results: ['745 Mbits/sec', '746 Mbits/sec']  
```

- Modify the flow entry with different queue.
```
curl -d '{"switch": "00:00:00:00:00:00:00:01", "name":"flow-mod-1", "cookie":"0", "priority":"32768","ingress-port":"2","active":"true", "actions":"enqueue=1:1"}' http://127.0.0.1:8080/wm/staticflowentrypusher/json
```
- Type "iperf h2 h1" in mininet   
```  
mininet> iperf h2 h1  
  *** Iperf: testing TCP bandwidth between h2 and h1  
  *** Results: ['50.2 Mbits/sec', '50.6 Mbits/sec']
```


