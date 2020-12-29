---
title: 手把手打造仿 mininet 網路
keywords: 'SDN,Network,Linux,Ubuntu,Mininet'
tags:
  - Mininet
  - SDN
  - Network
  - System
  - Ubuntu
abbrlink: 24071
date: 2017-06-23 03:03:19
---

相信不少開始學習 SDN 的人一定都有聽過 [mininet](http://mininet.org/) 這套軟體，甚至大量使用過。
之前於 [Mininet 運作原理](https://www.hwchiu.com/2014-08-19-mininet-parsing.html) 有分析過 `mininet` 的原始碼，並瞭解其運作原理。
而今天這篇文章要講述的是如何透過 **ip** 指令配上 **OpenvSwitch** 在自己的系統上建造出一個類似 **mininet** 的環境。

<!--more-->
# Environment
- Ubuntu 14.04 LTS
- OpenvSwitch (其實哪個版本都無所謂)
    - 先自行安裝好 ovs，並且將相關的 kernel module, daemons 都準備好。

# Goal
我們的目標是希望在系統上模擬一個最簡單的網路環境，包含了兩個終端的 **device** 的機器以及一台連接兩台機器的 **switch**。
由於我們只有一台實體的機器 (Ubuntu)，為了達成我們上述的目標，我們會使用 **OpenvSwitch** 安裝在 Ubuntu 上面作為一個 **software switch**，接下來我們要透過 **network namespace isolation** 的技術在 Ubuntu 上面創造兩個獨立的網路環境，分別代表兩個終端的 **device**

以圖片抽象來看，我們的目標大概就如下圖

![](http://i.imgur.com/02gIXfD.jpg)

以 **network namepsace** 創造出兩個獨立的網路環境，分別是 **ns1** 以及 **ns2**。
接下來以 **OpenvSwitch** 創造一個 **ovs-eth0** 的 switch
最後透過 **ip link** 的方式創造兩條虛擬的 **link** 將 **ovs-eth0** 與 **ns1/ns2** 給串起來，就可以形成一個簡單的網路拓樸。


# Experiment
## Step1
首先，我們要在系統上創建 **ove-eth0，關於 **openvswitch** 的安裝與啟動，本文就不再多敘述，網路上有滿多的文件都在講述其指令與教學。
因此這邊就直接使用 **ovs-vsctl** 該指令直接來創造我們所需要的 switch。

```bash
ovs-vsctl add-br ovs-eth0
ifconfig ovs-eth0 up
```

這時候系統上的架構就如下圖般，什麼都沒有，只有一個 switch。
![](http://i.imgur.com/A7BMUXD.jpg)

## Step2
上述已經弄好了 switch 後， 我們接下來要創立兩個獨立的網路空間 **network namespace**，這邊使用 **ip netns** 指令來幫我們達成。
先使用 **ip netns help** 來看看有那些指令可以使用。

```bash
>ip netns help
Usage: ip netns list
       ip netns add NAME
       ip netns delete NAME
       ip netns identify PID
       ip netns pids NAME
       ip netns exec NAME cmd ...
       ip netns monitor
```

在本範例中我們只會用到 **list**, **add** 以及 **exec** 兩個指令，就如同字面意思的意義一樣，用來創造檢視,創造 **network namespace** 以及在該 **netns** 內執行對應的指令。

依序執行下列指令，創造好這兩個 netns 後，我們可以透過 **ip netns list** 確認的確有產生兩個 netns 。

```bash
ip netns add ns1
ip netns add ns2
ip netns list
```

接下來我們可以透過 **ip netns exec ns1 bash** 這個指令在 **network namespace ns1** 內執行 **bash** 這個指令，這樣我們就可以暫時切換裡面。
```bash
ip netns exec ns1 bash
```

接下來直接執行 **ifconfig -a** 查看系統上面的網路資訊，你會發現什麼都不見了，只剩下一個最簡單的 **loopback** 介面。
```bash
> ifconfig -a
lo        Link encap:Local Loopback
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
這就是 **network isolation** 的功用，將網路完全隔絕開來，不過我們暫時還沒有任何連線可以使用，所以先執行 **exit** 離開該 **network namespace** 回到我們的 **Ubuntu**吧。

上述的指令執行完畢後，我們的系統大概如下圖，有一點點的長進了。
![](http://i.imgur.com/zzbxLwQ.jpg)

## Step3
我們已經將 **software switch** 以及相關的 **network namespace** 都準備好了，接下來我們要想辦法將這些東西串起來，打通整個連線。
這邊要使用的是 **ip link** 這個指令來處理，由於我們要在系統上創建的是一條虛擬的連結，稱之為 **veth**，這條虛擬連結要連接兩個 **interface**， 而這兩個 **interface** 則會分別給 **switch** 以及 **ns** 給使用，因此我們指令的原型大概如下

**ip link add name $name type veth peer name $name2**

上述指令代表會在系統中創建兩個 **interface**，名稱分別是 **$name** 以及 **$name2**，然後其中間透過 **veth** 方式串接起來，代表有任何封包從任何一端進入，都會從另外一端出來。

執行完下列指令後，可以透過 **ifconfig** 或是 **ip link** 看到剛創造出來的 **interface**

```bash
ip link add name vet-n1 type veth peer name ovs-1
ip link add name vet-n2 type veth peer name ovs-2
ifconfig vet-n1 up
ifconfig vet-n2 up
ifconfig ovs-1 up
ifconfig ovs-2 up
```

到這一步驟後，整個系統架構如下圖，已經有點樣子了，離目標只差一點點了。
![](http://i.imgur.com/Ek4X7S8.jpg)

## Step4
經過前述的所有準備，該有的東西都有了，剩下的就是將上述創建的 **interface** 給放到正確的地方上，並且配上一個相同網域的 **ip address**，就可以讓 **openvswitch** 以 **l2 briding** 的方式把封包給轉發了。

這邊我們要繼續 **ip** 指令，首先我們要將剛剛創建的 **vet-n1/vet-n2** 這兩張 **interface** 給丟到 **ns1/ns2** 裡面，指令如下。

**ip link set $interface netns $ns**， 套到我們的環境的話，就是

```bash
ip link set vet-n1 netns ns1
ip link set vet-n2 netns ns2
```

當執行完這些指令後，再度透過 **ip link** 你會發現 **vet-n1/vet-n2** 這兩張 **interface** 完全消失了，已經被從 **Ubuntu Host** 本身給搬移到上述創造好的 **network namespace** **n1/n2** 裡面了。

接下來我們使用 **ip netns exec** 指令進入到 **ns1/ns2** 裡面去設定我們的網路了。
我們有下列事情要做
- 將剛剛獲得到的 **vet-n1/vet-n2** 改名成 **eth0** (為了好看)
- 將 **eth0** 以及 **lo** 叫起來
- 幫 **eth0** 設定 ip 及網段。

所以指令大概如下

```bash
> ip netns exec ns1 bash
> ip link set vet-n1 name eth0
> ip addr add 10.0.0.101/24 dev eth0
> ip link set eth0 up
> ip link set lo up
> exit
```

上述的指令會將 **ns1** 相關的事情都處理完畢，這時候再針對 **ns2** 進行一樣的處理，唯一記得的是 **ip** 的部分記得不要重複即可。
一切完畢後，目前系統上的架構如下圖
![](http://i.imgur.com/gC3zpKs.jpg)

## Step5

最後回到 **Ubuntu(Host)** 本身，最後就剩下 **ovs-1/ovs-2** 這兩張 **interface** 還沒處理了。
這邊我們透過 **ovs-vsctl** 的指令，將該兩張 **interface** 都接到 **ovs-eth0** 上面即可。

```bash
ovs-vsctl add-port ovs-eth0 ovs-1
ovs-vsctl add-port ovs-eth0 ovs-2
ip link set ovs-eth0 up
ip link set ovs-1 up
ip link set ovs-2 up
```
一切大功告成，整個系統的架構就如一開始的目標一樣了。
![](http://i.imgur.com/02gIXfD.jpg)

這時候就可以透過 **ip netns exec ns2 ping 10.0.0.101** 類似的指令去確認 **ns1** 以及 **ns2** 能不能互通，更複雜一點還可以進去執行除了 **ping** 以外的指令。
若今天 **ovs-eth0** 也有將系統上其他的網卡也加入近來，更可以讓 **ns1/ns2** 與外界網路連通，唯一要注意的是由於我們沒有採用 **Controller** 近來處理，所以預設的 **openvswitch** 只會使用 **l2 briding** 的方式去轉送封包，因此不同網段的封包會不通的。

# Summary

本文中我們用了 **network namespace** 與 **openvswitch** 創造了一個類似 **mininet** 的環境，實際上 **mininet** 也是用一樣的方法去建置其模擬環境的。
我們除了學習怎麼使用這些工具外，若能對於其實作方法也有瞭解，更能夠幫助我們去思考該工具的極限以及其能力，同時也能夠加深我們自己的知識。
