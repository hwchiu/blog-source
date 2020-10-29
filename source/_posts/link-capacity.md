---
title: 'OpenFlow link capacity '
keywords: 'Openflow,Link,Capacity,Controller'
date: '2014-05-07 06:05'
comments: true
tags:
  - Openflow
  - SDN
  - Floodlight
  - Java
abbrlink: 5740
description:  在Openflow的協定中，有時候會想要知道每條`link`的`capacity`,然後就可以藉由當前的rate來判斷這個Link是否壅塞。 本文嘗試使用 `Floodlight` Controller 作為範例來展示如何使用透過預設的 API 來取得每個 Port 的資訊，並且從中計算出當前這條 Link 是否屬於壅塞或是閒置。 這類型的機制與資訊對於想要完成 Traffic Engineering 的開發者來說非常重要，畢竟這是其中一種可以幫每條連線加上權重的一種方式

---
在 [Openflow 1.0](http://archive.openflow.org/documents/openflow-spec-v1.0.0.pdf)中，對於每個Port的定義如下
```
/* Description of a physical port */
struct ofp_phy_port {
    uint16_t port_no;
    uint8_t hw_addr[OFP_ETH_ALEN];
    char name[OFP_MAX_PORT_NAME_LEN]; /* Null-terminated */
    uint32_t config; /* Bitmap of OFPPC_* flags. */
    uint32_t state; /* Bitmap of OFPPS_* flags. */
    /* Bitmaps of OFPPF_* that describe features. All bits zeroed if
    * unsupported or unavailable. */
    uint32_t curr; /* Current features. */
    uint32_t advertised; /* Features being advertised by the port. */
    uint32_t supported; /* Features supported by the port. */
    uint32_t peer; /* Features advertised by peer. */
};
```

<!--more-->



從這個定義之中，可以注意到有ㄧ個欄位叫做`curr`,他是一個bitmaps的表達式，由`ofp_port_features`所組成
```
/* Features of physical ports available in a datapath. */
    enum ofp_port_features {
    OFPPF_10MB_HD = 1 << 0, /* 10 Mb half-duplex rate support. */
    OFPPF_10MB_FD = 1 << 1, /* 10 Mb full-duplex rate support. */
    OFPPF_100MB_HD = 1 << 2, /* 100 Mb half-duplex rate support. */
    OFPPF_100MB_FD = 1 << 3, /* 100 Mb full-duplex rate support. */
    OFPPF_1GB_HD = 1 << 4, /* 1 Gb half-duplex rate support. */
    OFPPF_1GB_FD = 1 << 5, /* 1 Gb full-duplex rate support. */
    OFPPF_10GB_FD = 1 << 6, /* 10 Gb full-duplex rate support. */
    OFPPF_COPPER = 1 << 7, /* Copper medium. */
    OFPPF_FIBER = 1 << 8, /* Fiber medium. */
    OFPPF_AUTONEG = 1 << 9, /* Auto-negotiation. */
    OFPPF_PAUSE = 1 << 10, /* Pause. */
    OFPPF_PAUSE_ASYM = 1 << 11 /* Asymmetric pause. */
};
```

這樣看起來我們可以透過`features request`的方式來取得switch上每一個port的capacity，因此做了下列實驗

### 環境建置
-	Controller: **Floodlight**
-	Network environment: **mininet** or **OVS on PC**

#### Expr 1
- mn --controller=remote,ip=127.0.0.1, --topo tree,1
- curl http://127.0.0.1:8080/wm/core/switch/all/features/json
```
{
  "00:00:00:00:00:00:00:01": {
    "actions": 4095,
    "buffers": 256,
    "capabilities": 199,
    "datapathId": "00:00:00:00:00:00:00:01",
    "length": 176,
    "ports": [
      {
        "advertisedFeatures": 0,
        "config": 0,
        "currentFeatures": 192,
        "hardwareAddress": "32:38:53:8a:27:42",
        "name": "s1-eth1",
        "peerFeatures": 0,
        "portNumber": 1,
        "state": 0,
        "supportedFeatures": 0
      },
      {
        "advertisedFeatures": 0,
        "config": 0,
        "currentFeatures": 192,
        "hardwareAddress": "8a:5d:09:2f:cf:06",
        "name": "s1-eth2",
        "peerFeatures": 0,
        "portNumber": 2,
        "state": 0,
        "supportedFeatures": 0
      },
      {
        "advertisedFeatures": 0,
        "config": 0,
        "currentFeatures": 0,
        "hardwareAddress": "7e:00:4e:66:4d:45",
        "name": "s1",
        "peerFeatures": 0,
        "portNumber": 65534,
        "state": 0,
        "supportedFeatures": 0
      }
    ],
    "tables": -2,
    "type": "FEATURES_REPLY",
    "version": 1,
    "xid": 7
  }
}
```
- 從回傳的訊息中可以看到，除了lo以外的`currentFeatures`都是192，192就是2^7+2^6,所以對應到`ofp_port_features`就是`OFPPF_10GB_FD`以及`OFPPF_COPPER`

#### Expr 2
- 這次使用了`traffic control link`可調整頻寬的link來使用，看看是否會有所變化
- mn --controller=remote,ip=140.113.214.95,port=6633 --topo tree,1 --link tc,bw=100.0
- curl http://127.0.0.1:8080/wm/core/switch/all/features/json
```
{
  "00:00:00:00:00:00:00:01": {
    "actions": 4095,
    "buffers": 256,
    "capabilities": 199,
    "datapathId": "00:00:00:00:00:00:00:01",
    "length": 176,
    "ports": [
      {
        "advertisedFeatures": 0,
        "config": 0,
        "currentFeatures": 192,
        "hardwareAddress": "32:38:53:8a:27:42",
        "name": "s1-eth1",
        "peerFeatures": 0,
        "portNumber": 1,
        "state": 0,
        "supportedFeatures": 0
      },
      {
        "advertisedFeatures": 0,
        "config": 0,
        "currentFeatures": 192,
        "hardwareAddress": "8a:5d:09:2f:cf:06",
        "name": "s1-eth2",
        "peerFeatures": 0,
        "portNumber": 2,
        "state": 0,
        "supportedFeatures": 0
      },
      {
        "advertisedFeatures": 0,
        "config": 0,
        "currentFeatures": 0,
        "hardwareAddress": "7e:00:4e:66:4d:45",
        "name": "s1",
        "peerFeatures": 0,
        "portNumber": 65534,
        "state": 0,
        "supportedFeatures": 0
      }
    ],
    "tables": -2,
    "type": "FEATURES_REPLY",
    "version": 1,
    "xid": 7
  }
}
```
- 可以看到完全沒有變化，不管有沒有設定`tc link`,這個`currentFeatures`的值依然是固定在10G，因此就很好奇會不會是這個featureRequest本身並沒有實作出來，因此換一個網路環境再試試看


#### Expr 3
- 這次就不使用`mininet`而是直接用一台實體PC配上`OVS`來跑跑看
- curl http://127.0.0.1:8080/wm/core/switch/all/features/json
```
{
  "00:00:a0:36:9f:00:ed:04": {
    "actions": 4095,
    "buffers": 256,
    "capabilities": 199,
    "datapathId": "00:00:a0:36:9f:00:ed:04",
    "length": 272,
    "ports": [
      {
        "advertisedFeatures": 1711,
        "config": 0,
        "currentFeatures": 672,
        "hardwareAddress": "a0:36:9f:00:ed:06",
        "name": "eth3",
        "peerFeatures": 0,
        "portNumber": 3,
        "state": 0,
        "supportedFeatures": 1711
      },
      {
        "advertisedFeatures": 1711,
        "config": 0,
        "currentFeatures": 640,
        "hardwareAddress": "a0:36:9f:00:ed:05",
        "name": "eth2",
        "peerFeatures": 0,
        "portNumber": 2,
        "state": 1,
        "supportedFeatures": 1711
      },
      {
        "advertisedFeatures": 1711,
        "config": 0,
        "currentFeatures": 640,
        "hardwareAddress": "a0:36:9f:00:ed:07",
        "name": "eth4",
        "peerFeatures": 0,
        "portNumber": 4,
        "state": 1,
        "supportedFeatures": 1711
      },
      {
        "advertisedFeatures": 0,
        "config": 1,
        "currentFeatures": 0,
        "hardwareAddress": "a0:36:9f:00:ed:04",
        "name": "br0",
        "peerFeatures": 0,
        "portNumber": 65534,
        "state": 1,
        "supportedFeatures": 0
      },
      {
        "advertisedFeatures": 1711,
        "config": 0,
        "currentFeatures": 640,
        "hardwareAddress": "a0:36:9f:00:ed:04",
        "name": "eth1",
        "peerFeatures": 0,
        "portNumber": 1,
        "state": 1,
        "supportedFeatures": 1711
      }
    ],
    "tables": -1,
    "type": "FEATURES_REPLY",
    "version": 1,
    "xid": 4
  }
}

```
-	採用真正的網卡後，就會發現`currentFeatures`的值是有變化的，這代表`OVS`的確有實作這個功能，於是我就開始好奇，為什麼Mininet中得到的數值都是10G,tc link到底是什麼


#### Mininet
-	在仔細研究mininet的source code後，大致瞭解了整個運作流程
- 當`mininet`要在兩個switch間創造一條link的時候，是透過下列手段達成的
	-	`ip link add name s1-eth1 type veth peer name s2-eth1` 這種系統指令創造一個特殊的裝置`veth`，這兩個裝置的封包會彼此互通，因此就達成了`link`的功用

  -	此時透過`ethtool  s1-eth1`可以觀察到其中` Speed: 10000Mb/s`這樣的設定，他的速度就是設定成10G

  -	在`OVS`中會使用` struct ethtool_cmd`這種結構來獲取`port`的相關資訊，這時候他會根據`speed`這個欄位來設定`currentFeautres`的數值

  -	因此`mininet`創造出來的link預設都是10G，所以`OVS`那邊都會抓到10G的資訊

-	`traffic control`的部分則是透過系統的`tc`指令來做到速度限制的功能，所以不會動到每個port的設定


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