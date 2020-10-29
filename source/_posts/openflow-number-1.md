---
title: 'Openflow Introduction, Port Types'
keywords: 'openflow,port,type,introduction'
date: '2013-04-30 21:10'
comments: true
tags:
  - SDN
  - Openflow
abbrlink: 12741
description: 本文基於 Openflow 1.0 的規則書，跟大家分享一下在 Openflow 的規範裡到底什麼叫做 Port, 以及有多少種相關的 Port，在使用上要注意些什麼。

---


# Components

	|----------|
	|Controller|
	------------
	     |
	     | (openFlow protocol)
	--------------------|
	| OpenFlow | Group  |
	|  Channel | Table  |
	|----------|        |
	|  Flow      Flow   |
	| Table  -->Table   |
	|----------------------
   		(openflow switch)


Component是由controller跟openflow switch所組成的

每個openflow switch中，會利用openflow channel藉由openflow protocol跟controller溝通。

此外openflow switch中會含有至少一個的flow table，對於每個封包，會到flow table中去尋找對應的entry

來做相對應的事情，而group table中含有很多個group entries,每個group包含了一個action buckets的列表，

而action buckets則收集了各種action 以及相對應的參數。

## OpenFlow Ports
- openflow ports 是openflow switch彼此交換使用的port.
- 與 openflow switch 上真實的port不會完全對應.
- 封包進來的稱做為ingress port.
- 封包出去的稱為output port.
- 必須支援三種type的port
	- physical ports
	- logical ports
	- reserved ports

### Standard Ports
- 定義為physical ports,logical ports and LOCAL reserved port(不包含其他的reserved port)
### Physical Ports
- 由openflow switch定義
- 跟硬體上真實的port有關，對於一般的switch來說，是 one-to-one的關係
- 在openflow switch中，有些會使用virtualised的方式來管理ports,此時的physical ports就代表virtual slice
### Logical Ports
- 由openflow switch定義
- 跟硬體教無關，比physical port更高一階.
- 被用來使用一些 non-openflow methods(e.g. link aggregation groups, tunnels, loopback interfaces)
- 可封裝封包.
- 可對應到不同的physical ports.
- metadata中會含有Tunnel-ID.
### Reserved Ports
- 由specification 1.3定義
- 做一些通用的forwarding action (e.g. sending to controller,flooding, forwarding using non-OpenFlow methos)
- openflow switch 沒有要求要支援所有的reserved ports
- Required:(待釐清，有點模糊)
	- ALL:代表switch的所有port都可以用來forwarding 特定的封包，除了封包的ingress port跟被標記OFPPC_NO_FWD的port
	- CONTROLLER:可用在ingress port & output ports,用在output port時，會被封裝成pkcket-in message然後送往controller,當用在ingress port時，代表封包來自controller
	- TABLE:代表openflow pipeline的開始，只有當output action在packet-out message的action list中時才有效
	- IN_PORT:代表封包的ingress port, 只能在output port使用，會讓封包由ingress port送出去.
	- ANY:當沒有port被使用(wildcarded)時用在某些特殊的openflow command.
- Optional (openFlow-only switch 不支援)
	- NORMAL:傳統的switch處理方式
	- FLOOD:使用normal pipeline 來flooding.

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/JPwq4znr1

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=7EP1KY3

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