---
layout: post
title: 'Floodlight-modules-dependency'
date: 2013-06-30 21:21
comments: true
categories: [Openflow, Network, java]
tags:
	- SDN
	- Openflow
	- Network
	- Java
---
在floodlight這個openflow controller中，對於module之間的執行順序是如何決定的，這部分很重要




如圖為例，假設有四個component，分別是 LLDP、DEVICE、Forwarding、VirtualNetwork這四個module
如果今天switch送了一個封包到controller來，那這四個module誰要先處理這個封包? 順序交換是否會有影響?
<!--more-->

![](https://lh3.googleusercontent.com/-bzM04DIckB8/UdAltVq05LI/AAAAAAAAAsc/x6MLQsfi4YE/w1205-h491-no/1.jpg)

## FIFO ##
今天完全不考慮每個module之間的dependency，依照module被載入的順序來決定處理封包的順序
那我們就把所有進來的封包依照 LLDP->DEVICE->Forwarding->VirtualNetwork 這樣的順序去處理。
這邊要注意的是 

- Forwarding會把封包用最短路徑的方式傳送到destination
- VirtualNetwork會根據mac address建立一個layer 2的virtual network

假如依照FIFO的方式來處理封包

![](https://lh3.googleusercontent.com/-CC2tGU6YF7Q/UdAlt9GTt3I/AAAAAAAAAso/8TB5hWyxhfY/w1285-h762-no/2.jpg)


1. 封包先經過 ***Forwarding***決定如何轉送，並且把相關的flow entry送給對應的switch。

2. 封包在經過 ***VirtualNetwor***來決定如何處理，但此時已經沒有任何意義了，因為即使這邊發現封包的流向是不同VN要阻擋，但是先前的***forwarding***已經通知switch如何轉送，因此***VirtualNetwor***就變成雞肋了。

所以根據這個情形可以發現如果採用FIFO的模式，就必須要很仔細的設定每個module之間的關係，這樣當module數量過多時，會很麻煩，所以這不是一個很好的辦法。


## Priority ##
如果每個module都能夠設定一個優先度，然後依照優先度去排序得到一個運行的順序，那這樣每次撰寫新的module
只要設定一個優先度就好，不需要苦力的調整全部的順序。

這邊思考了一下，如果優先度採用數字的方式來比較，那一旦module變多的時候是否也要每個module都要做些調整，所以這部分一開始設定的時候就要想遠一點，避免未來的調整。

這邊介紹一下在floodlight中是如何決定module的運行順序的。
首先每個module必須要先override下列兩個function

```
isCallbackOrderingPrereq(String type, String name)
isCallbackOrderingPostreq(String type, String name)
```
第一個function代表 哪些module的哪些event要在我之前執行  
第二個function代表 哪些module的哪些event要在我之後執行

每個module之間就依靠這些function來決定 誰先誰後，因此假設今天四個module彼此的宣告如下
以下的type 都假定為 PACKET_IN。

### LLDP ###
- 不在乎誰在我前面
- 不在乎誰在我後面

### DEVICE ###
- LLDP 必須在我之前
- 不在乎誰在我後面

### Forwarding ###
- LLDP跟DEVICE 必須在我之前
- 不在乎誰在我後面

### VirtualNetwork ###
- LLDP跟DEVICE 必須在我之前
- Forwarding 必須在我之後

接下來有兩大步驟
1. 找尋terminal modules
2. 用terminal modules為起點跑DFS，建立modules的執行順序


***Terminal module***指的是其後面不會有任何module要執行的module，因此這種module可以拖延期執行次序，因為該module本身沒有限制一定要在哪裡執行。
### Algorithm (pseudo)
```
for(int i=0;i<modules.size;i++)
{	
  isTerminal = true;
  for(int j=0;j<modules.size;j++)
  {	 
    if( modules[j] go after modules[i])
    {
      isTerminal = false;
      break;
    }
  }
  if(isTerminal)
    terminalQueue.add(modules[i]);
}       
```

每個modules都去問其他的module，根據每個module先前定義的優先權function，如果所有的modules都沒有要求要在我之後那我就是terminal modules，反之只要有一個modules必須要在我之後執行，則就跳開。

接者針對每個terminal module都去跑一個DFS，來建立執行順序。
```
for(int i=0;i<terminalQueue.size();i++)
{
	visit(terminalQueue[i]);
}

function visit(listener)
{
	if(!visted.contain(listener)
  {
  	visted.add(listener)
    for(int i=0;i<modules.size();i++)
    {
    	if( modules[i] go before listener)
      	visit(modules[i])
    } 
    orderingQueue.add(listener);
  }
}

```

每次進入visit後，就去問其他的modules，看有沒有modules要在我前面執行，然後遞迴下去
如果這個modules都問完了，就把他加入到執行queue裡面。

以剛剛的範例來說，Terminal modules只會有一個modules 就是forwarding
然後以forwarding為起點去跑DFS，則過程如下


>  listener:forwarding
  visted:forwarding
  ordering:empty
  action:choose Device (因為forwarding 有宣示 DEVICE要在我之前)

---

>  listener:Device
  visted:forwarding,Device
  ordering:empty
  action:choose LLDP (因為DEVICE 有宣示 LLDP要在我之前)
  
--- 

>  listener:LLDP
  visted:forwarding,Device,LLDP
  ordering:LLDP
  action:找不到符合條件的modules,所以把自己加入到ordering。
  
---

>  listener:Device
  visted:forwarding,Device,LLDP
  ordering:LLDP,Device
  action:找不到符合條件的modules,所以把自己加入到ordering。
  
---

>  listener:forwarding
  visted:forwarding,Device,LLDP
  ordering:LLDP,Device
  action:choose VirtualNetwork (因為 VirtualNetwork 有宣示 forwarding 要在我之後)
  
---

>  listener:VirtualNetwork
  visted:forwarding,Device,LLDP,VirtualNetwork
  ordering:LLDP,Device,VirtualNetwork
  action:找不到符合條件的modules,所以把自己加入到ordering。 (因為DEVICE跟LLDP已經visted了，所以不會繼續跑)  
  
----
>  listener:forwarding
  visted:forwarding,Device,LLDP,VirtualNetwork
  ordering:LLDP,Device,VirtualNetwork,forwarding
  action:找不到符合條件的modules,所以把自己加入到ordering。 (因為其他都已經visted了，所以不會繼續跑)  
  
按照這個流程跑完，可以發現執行順序就是
LLDP,Device,VirtualNetwork,forwarding
符合我們的預期，同時這種設計可以讓module針對多個modules去進行相依性的處理。

但是這方面如果沒有寫好，就會造成dead lock，當發生deadlock時，就會找不到terminal modules，此時floodlight就會丟出錯誤。算是有做個錯誤預防。

modules dependency的部分就大概到這邊，有機會在看看nox & pox是如何處理這方面的。
