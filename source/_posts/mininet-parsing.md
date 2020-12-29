---
layout: post
title: Mininet 運作原理
date: '2014-08-19 12:54'
comments: true
tags:
  - SDN
  - Openflow
  - Mininet
  - Network
  - SourceCode
keywords: 'SDN,Mininet,Network,Openflow'
abbrlink: 5831
---
Introduction
------------
- 此篇文章的目標是用來釐清`mininet`是如何emulate網路中的**host**、**switch**以及**link**
- 主要以圖解、指令與mininet實際中的程式碼來描述所有的行為
- Mininet version: **2.1.0p1**
- Mininet目前是採用**network namespaces**來達到**network isolation**的功能，可以參考[這邊](http://mininet.org/overview/)的說明
- 底下會描述要如何在系統中仿真出一個如下圖般的拓樸

<!--more-->

![](https://lh3.googleusercontent.com/-Ks4Jnge1LM0/U_NVE6312vI/AAAAAAAABnc/UABXeIqFiUg/w804-h419-no/Figure0.png)


Steps
-----
一開始，我們的系統環境完全是空的，如下
![](https://lh4.googleusercontent.com/bHfBMKJ8lgNchmpK8mXEgxty3B609ChpBYx20qpYXso=w804-h419-no)

首先，我們要先為系統中創立兩個Host，在`mininet`裡面，每個Host其實就是一個`Node`的物件，可以在node.py中看到此物件的定義，如下。
``` python
class Node( object ):
    """A virtual network node is simply a shell in a network namespace.
       We communicate with it using pipes."""

    portBase = 0  # Nodes always start with eth0/port0, even in OF 1.0

    def __init__( self, name, inNamespace=True, **params ):
        """name: name of node
           inNamespace: in network namespace?
           params: Node parameters (see config() for details)"""

        # Make sure class actually works
        self.checkSetup()

        self.name = name
        self.inNamespace = inNamespace

        # Stash configuration parameters for future reference
        self.params = params

        self.intfs = {}  # dict of port numbers to interfaces
        self.ports = {}  # dict of interfaces to port numbers
                         # replace with Port objects, eventually ?
        self.nameToIntf = {}  # dict of interface names to Intfs

        # Make pylint happy
        ( self.shell, self.execed, self.pid, self.stdin, self.stdout,
            self.lastPid, self.lastCmd, self.pollOut ) = (
                None, None, None, None, None, None, None, None )
        self.waiting = False
        self.readbuf = ''

        # Start command interpreter shell
        self.startShell()
```

這邊可以看到，這邊會有一個變數**inNamespace**用來決定此Host是否要透過**network namespaces**來達到**network isolation**的功能，當一切變數都初始化後，就會呼叫**startShell()**來執行此Host。

``` python
    def startShell( self ):
        "Start a shell process for running commands"
        if self.shell:
            error( "%s: shell is already running" )
            return
        # mnexec: (c)lose descriptors, (d)etach from tty,
        # (p)rint pid, and run in (n)amespace
        opts = '-cdp'
        if self.inNamespace:
            opts += 'n'
        # bash -m: enable job control
        # -s: pass $* to shell, and make process easy to find in ps
        cmd = [ 'mnexec', opts, 'bash', '-ms', 'mininet:' + self.name ]
        self.shell = Popen( cmd, stdin=PIPE, stdout=PIPE, stderr=STDOUT,
                            close_fds=True )
        self.stdin = self.shell.stdin
        self.stdout = self.shell.stdout
        ...
```
這邊可以觀察到，`mininet`是透過一隻叫做`mnexec`的程式來執行，
### mnexec
- 透過參數-n來將此process給轉換到**network namespaces**中
- 程式內會透過**execvp**去執行參數中的指令，在此範例中該指令就是**"-ms mininet:"+self.name**。
	- 這邊可以透過執行mininet後，在執行**ps auxww | grep mininet**，應該會看到類似下面的結果
``` sh
root     22071  0.0  0.0  12308  1384 ?        Ss   23:10   0:00 bash -ms mininet:c0
root     22079  0.0  0.0  12308  1384 ?        Ss   23:10   0:00 bash -ms mininet:h1
root     22080  0.0  0.0  12308  1380 ?        Ss   23:10   0:00 bash -ms mininet:h2
root     22081  0.0  0.0  12308  1380 ?        Ss   23:10   0:00 bash -ms mininet:h3
root     22082  0.0  0.0  12308  1380 ?        Ss   23:10   0:00 bash -ms mininet:h4
root     22085  0.0  0.0  12312  1384 ?        Ss   23:10   0:00 bash -ms mininet:s1
root     22090  0.0  0.0  12312  1388 ?        Ss   23:10   0:00 bash -ms mininet:s2
root     22095  0.0  0.0  12312  1388 ?        Ss   23:10   0:00 bash -ms mininet:s3
root     22100  0.0  0.0  12312  1384 ?        Ss   23:10   0:00 bash -ms mininet:s4
root     22105  0.0  0.0  12312  1384 ?        Ss   23:10   0:00 bash -ms mininet:s5
```
- 並且把該`mnexec`的**stdout**,**stdin**給接起來，未來會需要透過這兩個FD來與該host溝通。

當初始化兩個Host後，系統中就會出現了兩個Host，且這兩個host都會透過**namespace**來達到**network isolation**，理論上我們要可以透過`ip netns show`來看到這些**namespaces**，實際上卻看不到，原因如同[此篇](https://mailman.stanford.edu/pipermail/mininet-discuss/2014-January/003796.html)所說。
此時，我們的系統如下
![](https://lh5.googleusercontent.com/oT14RIKBCPSRYLXLjhL8jC3vS5oPdMf67r5_gBRebuY=w804-h419-no)

創立好Host後，接下來要創立Switch，Switch有很多種選擇，包含了**OVSLegacyKernelSwitch**、**UserSwitch**、**OVSSwitch**，**IVSSwitch**此四種，一般常用的就是**OVSSwitch**
這四種Switch都繼承自**Switch**物件，而**Switch**物件則繼承自**Node**
-	Node
	- Switch
  	- OVSLegacyKernelSwitch
    - UserSwitch
    - OVSSwitch
    - IVSSwitch

在switch創立後，會透過**start**此function來進行相關初始化的動作，以OVSSwitch為例，就會執行一系列我們所熟悉的**ovs-***指令，如下。
``` python
    def start( self, controllers ):
        "Start up a new OVS OpenFlow switch using ovs-vsctl"
        if self.inNamespace:
            raise Exception(
                'OVS kernel switch does not work in a namespace' )
        # We should probably call config instead, but this
        # requires some rethinking...
        self.cmd( 'ifconfig lo up' )
        # Annoyingly, --if-exists option seems not to work
        self.cmd( 'ovs-vsctl del-br', self )
        self.cmd( 'ovs-vsctl add-br', self )
        if self.datapath == 'user':
            self.cmd( 'ovs-vsctl set bridge', self,'datapath_type=netdev' )
        int( self.dpid, 16 ) # DPID must be a hex string
        self.cmd( 'ovs-vsctl -- set Bridge', self,
                  'other_config:datapath-id=' + self.dpid )
        self.cmd( 'ovs-vsctl set-fail-mode', self, self.failMode )
        for intf in self.intfList():
            if not intf.IP():
                self.attach( intf )
        # Add controllers
        clist = ' '.join( [ 'tcp:%s:%d' % ( c.IP(), c.port )
                            for c in controllers ] )
        if self.listenPort:
            clist += ' ptcp:%s' % self.listenPort
        self.cmd( 'ovs-vsctl set-controller', self, clist )
        # Reconnect quickly to controllers (1s vs. 15s max_backoff)
        for uuid in self.controllerUUIDs():
            if uuid.count( '-' ) != 4:
                # Doesn't look like a UUID
                continue
            uuid = uuid.strip()
            self.cmd( 'ovs-vsctl set Controller', uuid,
                      'max_backoff=1000' )

```
在此程式中會去進行
1. 設定bridge
2. 設定datapath_type
3. 設定fail-mode
4. 設定controller

此時系統如下，系統中已經創立好了switch以及兩個host，這三個Node都分別透過**namespace**來達到了**network isolation**，只是彼此之間都尚未有任何Link存在。
![](https://lh4.googleusercontent.com/scsBAIEpo5-gO1k9tVPNTPHr-d7Q-Q3dgEm_nqhodYk=w804-h419-no)

接下來，會根據拓墣的Link情況去創建對應的Iterface。首先，這邊使用到**Link**這個物件來表示每一條Link，每個**Link**實際上對應到的是兩個**Node**上面的**Interface**。

``` python
class Link( object ):

    """A basic link is just a veth pair.
       Other types of links could be tunnels, link emulators, etc.."""

    def __init__( self, node1, node2, port1=None, port2=None,
                  intfName1=None, intfName2=None,
                  intf=Intf, cls1=None, cls2=None, params1=None,
                  params2=None ):
        ....
        if port1 is None:
            port1 = node1.newPort()
        if port2 is None:
            port2 = node2.newPort()
        if not intfName1:
            intfName1 = self.intfName( node1, port1 )
        if not intfName2:
            intfName2 = self.intfName( node2, port2 )

        self.makeIntfPair( intfName1, intfName2 )
```
這邊要觀察到的，**Link**物件會呼叫**makeIntfPair**此方法，此方法就可以將兩個**Interface**給串接起來

``` python
def makeIntfPair( intf1, intf2 ):
    """Make a veth pair connecting intf1 and intf2.
       intf1: string, interface
       intf2: string, interface
       returns: success boolean"""
    # Delete any old interfaces with the same names
    quietRun( 'ip link del ' + intf1 )
    quietRun( 'ip link del ' + intf2 )
    # Create new pair
    cmd = 'ip link add name ' + intf1 + ' type veth peer name ' + intf2
    return quietRun( cmd )
```
這邊可以看到，`mininet`實際上是透過系統中的`ip link`的方法將兩個**interface**創造一條**veth**的Link。
此時系統如下
![](https://lh6.googleusercontent.com/-oMP0TN5dms4/U_NZpcJaNuI/AAAAAAAABoI/O_zxCNeI27Y/w804-h419-no/Figure5.png)

接下來，我們要把這些**interface**給綁到特定的**Node**身上，在**Link**物件初始化後段，會去初始化兩個**Interface**真正的物件本體，
```
class Intf( object ):

    "Basic interface object that can configure itself."

    def __init__( self, name, node=None, port=None, link=None, **params ):
        """name: interface name (e.g. h1-eth0)
           node: owning node (where this intf most likely lives)
           link: parent link if we're part of a link
           other arguments are passed to config()"""
        self.node = node
        self.name = name
        self.link = link
        self.mac, self.ip, self.prefixLen = None, None, None
        # Add to node (and move ourselves if necessary )
        node.addIntf( self, port=port )
        # Save params for future reference
        self.params = params
        self.config( **params )
```
這邊要觀察的重點是每個**Interface**都會去呼叫**node.addIntf( self, port=port )**來處理，

```python
    def addIntf( self, intf, port=None ):
        """Add an interface.
           intf: interface
           port: port number (optional, typically OpenFlow port number)"""
        if port is None:
            port = self.newPort()
        self.intfs[ port ] = intf
        self.ports[ intf ] = port
        self.nameToIntf[ intf.name ] = intf
        debug( '\n' )
        debug( 'added intf %s:%d to node %s\n' % ( intf, port, self.name ) )
        if self.inNamespace:
            debug( 'moving', intf, 'into namespace for', self.name, '\n' )
            moveIntf( intf.name, self )
```
此方法最後會呼叫 **moveIntf** 來將該**interface**給處理，**moveIntf**則會呼叫**moveIntfNoRetry**將**Interface**給綁入到每個**Node**中。
``` python
def moveIntfNoRetry( intf, dstNode, srcNode=None, printError=False ):
    """Move interface to node, without retrying.
       intf: string, interface
        dstNode: destination Node
        srcNode: source Node or None (default) for root ns
        printError: if true, print error"""
    intf = str( intf )
    cmd = 'ip link set %s netns %s' % ( intf, dstNode.pid )
    if srcNode:
        srcNode.cmd( cmd )
    else:
        quietRun( cmd )
    links = dstNode.cmd( 'ip link show' )
    if not ( ' %s:' % intf ) in links:
        if printError:
            error( '*** Error: moveIntf: ' + intf +
                   ' not successfully moved to ' + dstNode.name + '\n' )
        return False
    return True
```
這邊可以看到，透過的指令則是**ip link set %s netns %s**，會將特定的**interface**給塞入特定**Node**的**namespace**之中
此時，我們的系統如下
![](https://lh5.googleusercontent.com/xn1vz7MvkXaGuCw3Dd_DYiHJ1nSZ992W_cZ6i-s7rmE=w804-h419-no)

## 最後
- 透過`ovs-vsctl add-port`將**Switch**上面的**Interface**都給OVS控管
``` python
    def attach( self, intf ):
        "Connect a data port"
        self.cmd( 'ovs-vsctl add-port', self, intf )
        self.cmd( 'ifconfig', intf, 'up' )
        self.TCReapply( intf )
```
- 設定Host上面網卡的**MAC**、**IP**、**Default Route**，此步驟會在Mininet噴出**Configuring hosts**時處理
``` python
    def config( self, mac=None, ip=None,
                defaultRoute=None, lo='up', **_params ):
        """Configure Node according to (optional) parameters:
           mac: MAC address for default interface
           ip: IP address for default interface
           ifconfig: arbitrary interface configuration
           Subclasses should override this method and call
           the parent class's config(**params)"""
        # If we were overriding this method, we would call
        # the superclass config method here as follows:
        # r = Parent.config( **_params )
        r = {}
        self.setParam( r, 'setMAC', mac=mac )
        self.setParam( r, 'setIP', ip=ip )
        self.setParam( r, 'setDefaultRoute', defaultRoute=defaultRoute )
        # This should be examined
        self.cmd( 'ifconfig lo ' + lo )
        return r
```
這三個指令最後都會呼叫到**sendCmd**來處理，此函式會利用先前執行``mnexec``得到的**stdin**,**stout**來與底下的host交換訊息。

以上就是一個`mininet`如何創造一個拓樸的簡單流程，有滿多細節都省略掉的，只挑重要的指令來看，來瞭解是如何透過系統指令來完成這些拓樸的。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=LG28Q5G

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
