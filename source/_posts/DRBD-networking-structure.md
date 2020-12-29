---
title: Drbd Networking Structure Introduction
tags:
  - DRBD
  - System
  - Networking
  - Kernel
  - SourceCode
keywords: 'DRBD,Kernel,DRBD9.0'
abbrlink: 57020
date: 2017-05-16 17:16:15
---

### Introduction
本文主要分析 **drbd** 在 **kernel space** 中關於 **networking** 這一部分用到的所有資料結構，這些資料結構主要分成兩個部分，一部分是通用的，一部分則是 **TCP** 連線專用的
#### 通用
- struct drbd_resource
- struct drbd_connection
- struct drbd_path
- struct drbd_listener
- struct drbd_transport
- struct drbd_transport_ops
- struct drbd_transport_class

#### TCP專用
- struct drbd_tcp_transport
- struct dtt_listener
- struct dtt_socket_container
- struct dtt_path

### Environment
- Drbd 9.0
- Using TCP as DRBD Transport

<!--more-->


接下來為了比較好理解整個過程，會先介紹每個結構在做什麼事情，然後在闡述這些結構之間的關係。
### 分析
#### 架構解釋
##### drbd_transport_class
**drbd_transport_class** 用來紀錄 networking module 相關資訊，譬如該 module 的名稱， 該 **kernel module**的 init function，其中 **instance_size** 以及 **path_instance_size** 則是用來記錄該網路實作過程中，繼承自 **drbd_tanasport_class**以及**drbd_path** 那些物件真正的大小。
以 TCP 舉例來說，他設計了一個物件 **drbd_tcp_transport**，裡面包含了 **drbd_transport_class** 以及一些 TCP 會用到的變數，這些零零總總加起來的總大小就是此 TCP module 真正要用到的大小。
這邊會這樣設計的原因是因為在更上層要透過 **kmalloc** 去要空間的時候，需要計算真正用到的大小，如下列應用
```c
3308     size = sizeof(*connection) - sizeof(connection->transport) + tc->instance_size;
```

```c
0194 struct drbd_transport_class {
0195     const char *name;
0196     const int instance_size;
0197     const int path_instance_size;
0198     struct module *module;
0199     int (*init)(struct drbd_transport *);
0200     struct list_head list;
0201 };
```

##### drbd_transport_ops
這個結構用來定義所有跟網路相關的操作，如 **connect**, **send** 等。
每個要實作Networking Module的 **kernel module**都必須要實做這些功能，並且設定好對應的 **function pointer**。
在 **drbd.ko**中，就會透過 **drbd_transport** 的方式去存取到這些對應的操作來使用，譬如
```c
0686     err = transport->ops->connect(transport);
```

```c
0130 struct drbd_transport_ops {
0131     void (*free)(struct drbd_transport *, enum drbd_tr_free_op free_op);
0132     int (*connect)(struct drbd_transport *);
0133
.........
0165     int (*recv)(struct drbd_transport *, enum drbd_stream, void **buf, size_t size, int flags);
........
0179     int (*recv_pages)(struct drbd_transport *, struct drbd_page_chain_head *, size_t size);
0180
0181     void (*stats)(struct drbd_transport *, struct drbd_transport_stats *stats);
0182     void (*set_rcvtimeo)(struct drbd_transport *, enum drbd_stream, long timeout);
0183     long (*get_rcvtimeo)(struct drbd_transport *, enum drbd_stream);
0184     int (*send_page)(struct drbd_transport *, enum drbd_stream, struct page *,
0185              int offset, size_t size, unsigned msg_flags);
0186     int (*send_zc_bio)(struct drbd_transport *, struct bio *bio);
0187     bool (*stream_ok)(struct drbd_transport *, enum drbd_stream);
0188     bool (*hint)(struct drbd_transport *, enum drbd_stream, enum drbd_tr_hints hint);
0189     void (*debugfs_show)(struct drbd_transport *, struct seq_file *m);
0190     int (*add_path)(struct drbd_transport *, struct drbd_path *path);
0191     int (*remove_path)(struct drbd_transport *, struct drbd_path *path);
0192 };
```

##### drbd_transport
真正用來抽象整個 networking module 的結構，將上面提到的 **drbd_transport_ops** 以及 **drbd_transport_class** 收錄到此結構中，最外層的 drbd 透過此物件可以呼叫到當前 networking 的實作方法。
``` c
0103 struct drbd_transport {
0104     struct drbd_transport_ops *ops;
0105     struct drbd_transport_class *class;
0106
0107     struct list_head paths;
0108
0109     const char *log_prefix;     /* resource name */
0110     struct net_conf *net_conf;  /* content protected by rcu */
0111
0112     /* These members are intended to be updated by the transport: */
0113     unsigned int ko_count;
0114     unsigned long flags;
0115 };
```

##### drbd_listener
接下來看到 **drbd_listener**，由於 **DRBD** 是分散式的架構，每個 **host** 同時是**clinet**也是 **server**，在扮演 **server** 的過程中，需要在本地上開啟一個 **socket** 並且透過 `listen`去處理該 socket 以接受之後的連線，這邊就用這個結構來儲存這相關的資訊。
就如同註解所說，這只是一個抽象概念而已，真正的實作則是依賴每個 networking model來處理，舉例來說， TCP module 則是包了一層 **dtt_listener**，裡面除了有最原始的 **drbd_listener** 之外，還放了 **TCP server** 使用的 **listen socket**。

``` c
0059 struct dtt_listener {
0060     struct drbd_listener listener;
0061     void (*original_sk_state_change)(struct sock *sk);
0062     struct socket *s_listen;
0063
0064     wait_queue_head_t wait; /* woken if a connection came in */
0065 };
```

``` c
0204 /* An "abstract base class" for transport implementations. I.e. it
0205    should be embedded into a transport specific representation of a
0206    listening "socket" */
0207 struct drbd_listener {
0208     struct kref kref;
0209     struct drbd_resource *resource;
0210     struct list_head list; /* link for resource->listeners */
0211     struct list_head waiters; /* list head for paths */
0212     spinlock_t waiters_lock;
0213     int pending_accepts;
0214     struct sockaddr_storage listen_addr;
0215     void (*destroy)(struct drbd_listener *);
0216 };
```
##### drbd_path
**drbd_path**的概念與 **drbd.conf** 中設定檔的概念相同，用來描述兩個 host 之間的連線，主要內容是 **ip(v4/v6):port**，所以結構中會有 **my_addr** 以及 **peer_addr**，用來紀錄兩端點的位址。
由於每個 **path** 中，本地端不但要當 client 連過去，同時也要當 server 等待對方連線，因此 **my_addr** 就會拿來當作 **listener** 使用，所以可以看到該成員有一個指向 **drbd_listener** 的指標 **listener**。

``` c
0085 struct drbd_path {
0086     struct sockaddr_storage my_addr;
0087     struct sockaddr_storage peer_addr;
0088
0089     struct kref kref;
0090
0091     int my_addr_len;
0092     int peer_addr_len;
0093     bool established; /* updated by the transport */
0094
0095     struct list_head list; /* paths of a connection */
0096     struct list_head listener_link; /* paths waiting for an incomming connection,
0097                        head is in a drbd_listener */
0098     struct drbd_listener *listener;
0099 };
```

##### drbd_connection
**drbd_connection** 的概念與 **drbd.conf** 中 **connection**的描述相同，相對於 **path** 來說是個更高一等的抽象概念，描述兩個 host 之間的連線，
這些連線是由很多個 **path** 所組成的所以一個 **connection** 可以有很多條 **path** 而**connection** 中間的傳輸會透過其中一條 **path** 來交換，對 **connection** 來說，同時只會使用一條 **path** 傳輸，並沒有辦法達到 **link aggregation**的功效。

下圖結構中可以觀察到有一個 **transport**，每條 **connection** 都會綁定一個 **transport** 的物件，該條 connection 的所有操作都依照該物件內容去執行，所以在此架構下，是可以做到每條 **connection** 使用不同的傳輸方法，譬如原生TCP或是自行實作的物件。

``` c
0904 struct drbd_connection {
0905     struct list_head connections;
..................
1047
1048     unsigned int peer_node_id;
1049     struct list_head twopc_parent_list;
1050     struct drbd_transport transport; /* The transport needs to be the last member. The acutal
1051                         implementation might have more members than the
1052                         abstract one. */
1053 };
```

##### struct dtt_path
這邊除了將本來的 **drbd_path** 包起來外，還多了一個list來處理上述 **dtt_socket_container** 的物件，這邊目前沒有辦法理解為什麼需要用list來保存，我以為只需要用一個 socket 的物件就可以了。
``` c
0076 struct dtt_path {
0077     struct drbd_path path;
0078
0079     struct list_head sockets; /* sockets passed to me by other receiver threads */
0080 };
```
##### struct dtt_socket_container
此物件還令人滿納悶的，目前還沒有想到什麼情況下會需要這個東西...，待之後突然領悟了再回來補足。
``` c
0071 struct dtt_socket_container {
0072     struct list_head list;
0073     struct socket *socket;
0074 };
```

##### struct dtt_listener
真正實作 **drbd_listener** 的物件，由於是走 TCP 的架構，所以需要一個 **socket** 來進行 listen 的動作。
其餘的成員 **original_sk_state_change**以及 **wait** 待詳細分析 tcp 程式碼時再說明。
``` c
0059 struct dtt_listener {
0060     struct drbd_listener listener;
0061     void (*original_sk_state_change)(struct sock *sk);
0062     struct socket *s_listen;
0063
0064     wait_queue_head_t wait; /* woken if a connection came in */
0065 };
```

##### struct drbd_tcp_transport
這邊則是 TCP 方面對於 **drbd_transport** 的實現，由於要支援 **DATA_STREAM** 以及 **CONTROL_STREAM**，所以這邊每個 tcp_transport 則是會使用兩個 **sockets** 的物件來保存。
``` c
0051 struct drbd_tcp_transport {
0052     struct drbd_transport transport; /* Must be first! */
0053     spinlock_t paths_lock;
0054     unsigned long flags;
0055     struct socket *stream[2];
0056     struct buffer rbuf[2];
0057 };
```

##### drbd_resource
在 kernel 層級這邊，最主要的管理者是 **drbd_resource**，他控管所有的資源，包含了 listener, connections等與網路相關的資訊。

可以看到 **struct drbd_resource** 中的成員有兩個 **list_head** 的成員，分別用來將 **connection** 以及 **listener** 給串起來。
connection(**drbd_connection**) 代表的就是每一對 host 的 connection，如同在 **user-space** 中 **drbd.conf** 中設定的那樣，
```c
0820 struct drbd_resource {
...
0832     struct list_head connections;
....
0893     struct list_head listeners;
...
```

#### 架構圖
##### Connection and Path
抽象來說，每個 **connection**會擁有多條 **path**，但是實際上其實是每條 **connection**內的 **drbd_transport** 擁有多條 **path**。
為了容易理解，所以底下都使用 **connection** 取代 **connection**內部的 **drbd_transport**。

每一個 **connection** 會使用 **double link list** 的方式來維護多條 **path**，所以在 **connetion** 中會有一個 **list head**的物件來指向該 **link list**的第一個。
如下圖
![](http://i.imgur.com/udR9bVq.jpg)

##### Listener and Path
每一個 **listener** 都代表一個 listen socket，而不同的 **path** 則可以擁有相同的 **listen socket**，因為只會有其中一個真正被使用到，所以在架構上每個 **listener** 也會有一個 **double link list** 串起用到本身的 **path**。
如下圖
![](http://i.imgur.com/CZuF7on.jpg)

##### Resource and Conection/Listener
Resource是最上層的物件，掌管所有的 connection，因此也會使用 **double link list**去掌管所有的 connections。
此外，為了在某些步驟能夠更快速的查找所有的 listener， resource 本身也用了一個 **double link list** 串起所有的 resource。
將上述這些所有結果都繪成圖片，並且將所有的 **double link** 都簡化成 **single link** 且透過不同的箭頭符號表現不同的 **link type**。則結果會如下圖。

![relation](http://i.imgur.com/QVQ3D2B.jpg)

### 結論
- drbd 9.0 之後為了支援 [multiple path](http://lists.linbit.com/pipermail/drbd-announce/2016-February/000218.html)，在架構上有不小的改動，所以整個 path 相關的結構就變得複雜許多。
- 本篇主要著重在抽象層的概念上，實作上還有許多小細節沒有提起，這些必須要到分析 TCP 模組實際上是怎麼運作時才有更深的理解與體悟。
