---
title: DRBD v9.0 Network Work Flow(ii)
date: 2017-05-12 17:22:18
tags:
	- System
	- DRBD
	- Network
keywords: DRBD
---
Introduction
============
本文延續之前研究 drbd 9.0 網路的工作流程，這篇文章主要在研究其 kernel space 中的行為與邏輯。

從之前 `drbdsetup` 那邊可以觀察到，這三個指令的結構如下，其中要特別注意的就是
**DRBD_ADM_CONNECT**, **DRBD_ADM_NEW_PEER** 以及 **DRBD_ADM_NEW_PATH**。
這三個數值其實是給 netlink 使用的，在 kernel 端會去註冊遇到三種類型的 netlink 應該要怎麼處理。
```c
0397     {"connect", CTX_PEER_NODE,
0398         DRBD_ADM_CONNECT, DRBD_NLA_CONNECT_PARMS,
0399         F_CONFIG_CMD,
0400      .ctx = &connect_cmd_ctx,
0401      .summary = "Attempt to (re)establish a replication link to a peer host." },
```
```c
0403     {"new-peer", CTX_PEER_NODE,
0404         DRBD_ADM_NEW_PEER, DRBD_NLA_NET_CONF,
0405         F_CONFIG_CMD,
0406      .ctx = &new_peer_cmd_ctx,
0407      .summary = "Make a peer host known to a resource." },
```
```c
0415     {"new-path", CTX_PEER_NODE,
0416         DRBD_ADM_NEW_PATH, DRBD_NLA_PATH_PARMS,
0417         F_CONFIG_CMD,
0418      .drbd_args = (struct drbd_argument[]) {
0419         { "local-addr", T_my_addr, conv_addr },
0420         { "remote-addr", T_peer_addr, conv_addr },
0421         { } },
0422      .ctx = &path_cmd_ctx,
0423      .summary = "Add a path (endpoint address pair) where a peer host should be reachable." },
```

<!--more-->

在 `drbd_genl.h` 中可以看到有下列程式碼，這邊主要是註冊 netlink 的 MACRO，這邊注意的則是 **GENL_doit**，裡面放的是一個 fptr，指向當此 type 被觸發後，要用來處理的 function。
所以我們可以明確的知道，在 kernel裡面對應三個指令的 function 分別是
**drbd_adm_new_peer**, **drbd_adm_new_path**, 以及 **drbd_adm_connect**。
```c
0350 GENL_op(DRBD_ADM_NEW_PEER, 44, GENL_doit(drbd_adm_new_peer),
0351     GENL_tla_expected(DRBD_NLA_CFG_CONTEXT, DRBD_F_REQUIRED)
0352     GENL_tla_expected(DRBD_NLA_NET_CONF, DRBD_GENLA_F_MANDATORY)
0353 )
0354 
0355 GENL_op(DRBD_ADM_NEW_PATH, 45, GENL_doit(drbd_adm_new_path),
0356     GENL_tla_expected(DRBD_NLA_CFG_CONTEXT, DRBD_F_REQUIRED)
0357     GENL_tla_expected(DRBD_NLA_PATH_PARMS, DRBD_F_REQUIRED)
0358 )
0359 
0369 
0370 GENL_op(DRBD_ADM_CONNECT, 10, GENL_doit(drbd_adm_connect),
0371     GENL_tla_expected(DRBD_NLA_CFG_CONTEXT, DRBD_F_REQUIRED)
0372     GENL_tla_expected(DRBD_NLA_CONNECT_PARMS, DRBD_GENLA_F_MANDATORY)
0373 )
```


drbd_adm_new_peer
=================
基本上 peer 跟 connection 是差不多的東西的，所以這個 function 其實就是創好一條 connection，這邊的 connection 是個抽象層的概念，並不代表底下實際上的網路連線已經建立完畢了。

如果此 connection 之前已經創立過，則 **adm_ctx.connection** 該指標就會指向該 **connection**，否則就透過 `adm_new_connection` 創立一條 connection。

```c
3656 int drbd_adm_new_peer(struct sk_buff *skb, struct genl_info *info)
3657 {
3658     struct drbd_config_context adm_ctx;
3659     struct drbd_connection *connection;
3660     enum drbd_ret_code retcode;
3661 
3662     retcode = drbd_adm_prepare(&adm_ctx, skb, info, DRBD_ADM_NEED_PEER_NODE);
3663     if (!adm_ctx.reply_skb)
3664         return retcode;
3665 
3666     mutex_lock(&adm_ctx.resource->adm_mutex);
3667 
3668     if (adm_ctx.connection) {
3669         retcode = ERR_INVALID_REQUEST;
3670         drbd_msg_put_info(adm_ctx.reply_skb, "peer connection already exists");
3671     } else {
3672         retcode = adm_new_connection(&connection, &adm_ctx, info);
3673     }
3674 
3675     mutex_unlock(&adm_ctx.resource->adm_mutex);
3676     drbd_adm_finish(&adm_ctx, info, retcode);
3677     return 0;
3678 }
```
## adm_new_connection
首先，先確認當前還沒有 connection 存在，接下來我們要開始取得一些跟 network 相關的設定，所以這邊會先透過 `kzalloc` 在 kernel 內產生一個空間，接下來透過 `net_conf_from_attrs` 從 **netlink** 的 **attribute**中讀取相關的資料，然後 **new_net_conf**  結構中，由於 `net_conf_from_attrs` 是支由 MACRO 展開的 function，內容不好閱讀，只要知道能夠從 **netlink** 內讀取到想要的數據，並且拿出來即可。
```c
3326     *ret_conn = NULL;
3327     if (adm_ctx->connection) {
3328         drbd_err(adm_ctx->resource, "Connection for peer node id %d already exists\n",
3329              adm_ctx->peer_node_id);
3330         return ERR_INVALID_REQUEST;
3331     }
3332 
3333     /* allocation not in the IO path, drbdsetup / netlink process context */
3334     new_net_conf = kzalloc(sizeof(*new_net_conf), GFP_KERNEL);
3335     if (!new_net_conf)
3336         return ERR_NOMEM;
3337 
3338     set_net_conf_defaults(new_net_conf);
3339 
3340     err = net_conf_from_attrs(new_net_conf, info);
3341     if (err) {
3342         retcode = ERR_MANDATORY_TAG;
3343         drbd_msg_put_info(adm_ctx->reply_skb, from_attrs_err_to_txt(err));
3344         goto fail;
3345     }
```
接下來會從設定檔中判斷當前的網路連線是走什麼協定，一般免費社群使用的版本只有 tcp 可以，接洽購買後可以獲得 RDMA 相關的 kernel module 來使用。
所以這邊最後會透過 `drbd_get_transport_class` 根據對應的名稱來找到對應的 netowrk module 實作。
``` c
3347     transport_name = new_net_conf->transport_name[0] ? new_net_conf->transport_name : "tcp";
3348     tr_class = drbd_get_transport_class(transport_name);
3349     if (!tr_class) {
3350         retcode = ERR_CREATE_TRANSPORT;
3351         goto fail;
3352     }
```

接下來則是透過 `drbd_create_connection` 來創建 connection，這邊會將剛剛得到的 **transport_class** 一併傳入，因為最後會需要該 **transport_class** 去執行底層的 init。

```c
3354     connection = drbd_create_connection(adm_ctx->resource, tr_class);
3355     if (!connection) {
3356         retcode = ERR_NOMEM;
3357         goto fail_put_transport;
3358     }
```

## drbd_create_connection
一開始，就先透過 `kzalloc` 去創建一個空間供 **connection**使用，這邊可以注意到 **size** 的算法非常特別，除了直接用 `sizeof` 算出該物件外，最後會有一個大小的微調
**- sizeof(connection->transport) + tc->instance_size**
這邊原因要牽扯到 **drbd_connection** 的實作內容，在其架構中有這樣一段註解
```c
1050     struct drbd_transport transport; /* The transport needs to be the last member. The acutal
1051                         implementation might have more members than the
1052                         abstract one. */
1053 };
```
可以看到其實最後一個欄位算是一個比較抽象的概念，實際上底層的實作可以有更多的變化，所以這邊在計算真正整體大小時，要先扣掉 **sizeof drbd_transport**，然後加上該實作真正用到的大小 **tc->instance_size**。

```c
3302 struct drbd_connection *drbd_create_connection(struct drbd_resource *resource,
3303                            struct drbd_transport_class *tc)
3304 {
3305     struct drbd_connection *connection;
3306     int size;
3307 
3308     size = sizeof(*connection) - sizeof(connection->transport) + tc->instance_size;
3309     connection = kzalloc(size, GFP_KERNEL);

```
接下來就要開始初始化 **drbd__connection** 內部的各種結構，包含各種 link list 相關的結構。
在一切初始化完畢後，最後呼叫 **transport class** 自己本身的 **init**。
```c
3374     if (tc->init(&connection->transport))
3375         goto fail;
```

在創建完畢 connection 後，接下來針對 **net_option**，**crtpyo**， **peer device** 去進行初始化的動作，
中間有一段則是將該 connetion 給加到 ** resource ** 此物件中，用 link list 的方式把所有的 connection 都綁起來，未來有其他指令要找到 connection 要使用時，就可以透過此方式找到之前創建的 connection
```c
3401     spin_lock_irq(&adm_ctx->resource->req_lock);
3402     list_add_tail_rcu(&connection->connections, &adm_ctx->resource->connections);
3403     spin_unlock_irq(&adm_ctx->resource->req_lock);
```
最後呼叫`drbd_thread_start` 去創建一個 **kernel thread**來運行 `drbd_sender`　此 thread。
```c
3467     drbd_thread_start(&connection->sender);
```
大致上此 function 就結束了。
整個**drbd_adm_new_peer**執行完畢後， kernel 內的 **resource** 底下就會有一個 **drbd_connection**的物件在運行，接下來的指令都會嘗試透過 `drbd_get_connection_by_node_id` 的方式得到該 connection 來進行後續操作。

drbd_adm_new_path
=================
在透過 `drbd_adm_new_peer` 創立一個 connection (peer) 後，接下來我們要在這條    connection 上創立一個新的 path， path 代表的就是實際上連線會對應的 ip address 以及對應的 port。
一開始會先透過 `drbd_adm_prepare` 進行一些資源的獲取，包含 **connection** 也會在裡面取得，然後放到 **adm_ctx.connection** 變數上。
接下來就透過 `adm_add_path` 進行細部的處理。
```c
3680 int drbd_adm_new_path(struct sk_buff *skb, struct genl_info *info)
3681 {
3682     struct drbd_config_context adm_ctx;
3683     enum drbd_ret_code retcode;
3684 
3685     retcode = drbd_adm_prepare(&adm_ctx, skb, info, DRBD_ADM_NEED_CONNECTION);
3686     if (!adm_ctx.reply_skb)
3687         return retcode;
3688 
3689     /* remote transport endpoints need to be globaly unique */
3690     mutex_lock(&adm_ctx.resource->adm_mutex);
3691 
3692     retcode = adm_add_path(&adm_ctx, info);
3693 
3694     mutex_unlock(&adm_ctx.resource->adm_mutex);
3695     drbd_adm_finish(&adm_ctx, info, retcode);
3696     return 0;
3697 }
```
## adm_add_path
首先先從 **connection** 中取得對應的 **drbd_transport** 的實作，不過這邊都沒有任何檢查，所以如果今天還沒有執行 `add_peer` 前就先執行 `add_path`，可能會有 **Null pointer dereferences** 的問題。

```c
3538 static enum drbd_ret_code
3539 adm_add_path(struct drbd_config_context *adm_ctx,  struct genl_info *info)
3540 {
3541     struct drbd_transport *transport = &adm_ctx->connection->transport;
3542     struct nlattr *my_addr = NULL, *peer_addr = NULL;
3543     struct drbd_path *path;
3544     enum drbd_ret_code retcode;
3545     int err;
```

接下來就如同上述的步驟一樣，先從 **netlink** 中取出我們需要的資訊，在這個指令中，我們需要的是一條 path 兩端點的 **address(ip:port)**。接者透過`check_path_usable`檢查該參數，譬如是否存在，是否已經使用過。
```c
3547     /* parse and validate only */
3548     err = path_parms_from_attrs(NULL, info);
3549     if (err) {
3550         drbd_msg_put_info(adm_ctx->reply_skb, from_attrs_err_to_txt(err));
3551         return ERR_MANDATORY_TAG;
3552     }
3553     my_addr = nested_attr_tb[__nla_type(T_my_addr)];
3554     peer_addr = nested_attr_tb[__nla_type(T_peer_addr)];
```
一切準備完畢後，開始創立 **strcut drbd_path**，先從 kernel 要空間，接下來把兩端點的 address 都複製進去，最後就讓 **transport class** 自行去負責要怎麼處理了，於是呼叫了 `transport->ops->add_path` 去處理。 在本文的範例中使用的是 TCP 的方式，最後則是透過 `dtt_add_path` 去處理，詳細處理的流程之後會再仔細研究整個 TCP 層的架構。
```c
3562     path = kzalloc(transport->class->path_instance_size, GFP_KERNEL);
3563     if (!path)
3564         return ERR_NOMEM;
3565 
3566     path->my_addr_len = nla_len(my_addr);
3567     memcpy(&path->my_addr, nla_data(my_addr), path->my_addr_len);
3568     path->peer_addr_len = nla_len(peer_addr);
3569     memcpy(&path->peer_addr, nla_data(peer_addr), path->peer_addr_len);
3570 
3571     kref_init(&path->kref);
3572 
3573     err = transport->ops->add_path(transport, path);
3574     if (err) {
3575         kref_put(&path->kref, drbd_destroy_path);
3576         drbd_err(adm_ctx->connection, "add_path() failed with %d\n", err);
3577         drbd_msg_put_info(adm_ctx->reply_skb, "add_path on transport failed");
3578         return ERR_INVALID_REQUEST;
3579     }
```
drbd_adm_connect
================
一切都準備完畢後，接下來就可以透過 `drbd_adm_connect` 真正地建立起兩端的連線。如同慣例，一開始都會先呼叫 `drbd_adm_prepare` 進行資源的整理，接下來就可以直接從 **adm_ctx.connection** 去取得先前創立的連線物件，然後判斷該連線目前的狀態。
當初創建好連線時，預設的狀態就是 **C_STANDALONE**。
```c
3584 int drbd_adm_connect(struct sk_buff *skb, struct genl_info *info)
3585 {
3586     struct drbd_config_context adm_ctx;
3587     struct connect_parms parms = { 0, };
3588     struct drbd_peer_device *peer_device;
3589     struct drbd_connection *connection;
3590     enum drbd_ret_code retcode;
3591     enum drbd_conn_state cstate;
3592     int i, err;
3593 
3594     retcode = drbd_adm_prepare(&adm_ctx, skb, info, DRBD_ADM_NEED_CONNECTION);
3595     if (!adm_ctx.reply_skb)
3596         return retcode;
3597 
3598     connection = adm_ctx.connection;
3599     cstate = connection->cstate[NOW];
3600     if (cstate != C_STANDALONE) {
3601         retcode = ERR_NET_CONFIGURED;
3602         goto out;
3603     }
```
接下來透過 `first_path` 確認該條 **connection** 至少有一條 **path** 存在，因為一個 **connection** 可以有多條 **path**，且這些 **path** 是透過 **link list** 的方式去紀錄的，所以只要判斷該 **list** 的第一個就知道目前有沒有至少一條 **path** 存在。
```c
3605     if (first_path(connection) == NULL) {
3606         drbd_msg_put_info(adm_ctx.reply_skb, "connection endpoint(s) missing");
3607         retcode = ERR_INVALID_REQUEST;
3608         goto out;
3609     }
```
最後透過 `change_cstate` 的方式來改變當前的狀態，然後透過一連串的呼叫宇改變，最後會在`drbd_receive` 內呼叫起 `conn_connect` 來進行真正的連線。
這中間的過程就不詳細描述，用兩張簡單的圖片大致說明即可。
首先透過第一張圖的流程，最後會跑到 `queue_after_state_change_work` 裡面，在裡面會創建一個 **work**，然後這個 **work** 裡面的 **call back function**會指向 **w_after_state_change**，最後把該 **work** 透過 `drbd_queue_work` 放入 resource 內的 **work list**。

![flow](http://i.imgur.com/xg6LxPS.jpg)
```c
1901 static void queue_after_state_change_work(struct drbd_resource *resource,
1902                       struct completion *done)
1903 {
1904     /* Caller holds req_lock */
1905     struct after_state_change_work *work;
1906     gfp_t gfp = GFP_ATOMIC;
1907 
1908     work = kmalloc(sizeof(*work), gfp);
1909     if (work)
1910         work->state_change = remember_state_change(resource, gfp);
1911     if (work && work->state_change) {
1912         work->w.cb = w_after_state_change;
1913         work->done = done;
1914         drbd_queue_work(&resource->work, &work->w);
1915     } else {
1916         kfree(work);
1917         drbd_err(resource, "Could not allocate after state change work\n");
1918         if (done)
1919             complete(done);
1920     }
1921 }
```

接下來如下圖，當 **resource** 一開始透過創立 **resource** 時，就會叫一起一隻 **kernel thread**，會專注於執行 `drbd_worker` 這隻 function，而這 function 內部則會不斷的把 **resource** 內部的 works 給拿出來執行。

![flow-2](http://i.imgur.com/Mya9Xss.jpg)

```c
2746 int drbd_worker(struct drbd_thread *thi)
2747 {
2748     LIST_HEAD(work_list);
2749     struct drbd_resource *resource = thi->resource;
2750     struct drbd_work *w;
2751 
2752     while (get_t_state(thi) == RUNNING) {
2753         drbd_thread_current_set_cpu(thi);
.................
2793 
2794         while (!list_empty(&work_list)) {
2795             w = list_first_entry(&work_list, struct drbd_work, list);
2796             list_del_init(&w->list);
2797             update_worker_timing_details(resource, w->cb);
2798             w->cb(w, 0);
2799         }
2800     }
......................
2826     return 0;
2827 }
```
最後呼叫該 **work** 的 **call back function**，最後會執行到 `w_after_state_change`，在這個 **function**內，最後會去把每個 connection 內部的 **kernel thread** 給叫起來，而這隻 **kernel thread** 則會呼叫 **drbd_receiver**

```c
2782 static int w_after_state_change(struct drbd_work *w, int unused)
2783 {
............
3247     for (n_connection = 0; n_connection < state_change->n_connections; n_connection++) {
............
3254         /* Upon network configuration, we need to start the receiver */
3255         if (cstate[OLD] == C_STANDALONE && cstate[NEW] == C_UNCONNECTED)
3256             drbd_thread_start(&connection->receiver);
3257 
............
3266     }
......
3287     return 0;
3288 }

```
當 **kernel thread** 起來後，接下來就會呼叫 `conn_connect` 來進行後續的連線，當連線成功後，就會呼叫 `drbdd` 進入 **while loop** 內來處理。
所以接下來就繼續來觀察 **conn_connect** 底下到底怎麼做。
```c
7646 int drbd_receiver(struct drbd_thread *thi)
7647 {
7648     struct drbd_connection *connection = thi->connection;
7649
7650     if (conn_connect(connection)) {
7651         blk_start_plug(&connection->receiver_plug);
7652         drbdd(connection);
7653         blk_finish_plug(&connection->receiver_plug);
7654     }
7655
7656     conn_disconnect(connection);
7657     return 0;
7658 }
```
```c 
7081 static void drbdd(struct drbd_connection *connection)
7082 {
7087     while (get_t_state(&connection->receiver) == RUNNING) {
7088         struct data_cmd const *cmd;
7089 
7090         drbd_thread_current_set_cpu(&connection->receiver);
7091         update_receiver_timing_details(connection, drbd_recv_header_maybe_unplug);
7092         if (drbd_recv_header_maybe_unplug(connection, &pi))
7093             goto err_out;
7094 
7095         cmd = &drbd_cmd_handler[pi.cmd];
7096         if (unlikely(pi.cmd >= ARRAY_SIZE(drbd_cmd_handler) || !cmd->fn)) {
7097             drbd_err(connection, "Unexpected data packet %s (0x%04x)",
7098                  drbd_packet_name(pi.cmd), pi.cmd);
7099             goto err_out;
7100         }
.............
7131     }
...
7136 }
```
## conn_connect
首先，一開始會先設定當前的 protocol version，主要是用來區分 drbd8 以及 drbd9 用的，預設先當作 drbd 8 (version 80)。接下來則會改變當前的狀態，將 **C_STANDALONE** 轉換成 **C_CONNECTING**。
最後就呼叫 **transport class**去執行自己實作的 **connect**。

```c
0665 static bool conn_connect(struct drbd_connection *connection)
0666 {
................
0675 start:
0676     have_mutex = false;
0677     clear_bit(DISCONNECT_EXPECTED, &connection->flags);
0678     if (change_cstate(connection, C_CONNECTING, CS_VERBOSE) < SS_SUCCESS) {
0679         /* We do not have a network config. */
0680         return false;
0681     }
0682 
0683     /* Assume that the peer only understands protocol 80 until we know better.  */
0684     connection->agreed_pro_version = 80;
0685 
0686     err = transport->ops->connect(transport);
0687     if (err == -EAGAIN) {
0688         if (connection->cstate[NOW] == C_DISCONNECTING)
0689             return false;
0690         goto retry;
0691     } else if (err < 0) {
0692         drbd_warn(connection, "Failed to initiate connection, err=%d\n", err);
0693         goto abort;
0694     }
```
接下來去設定每個 **socket** 的 send/recevie timeout，詳細的用途可以參考[SO_RCVTIMEO and SO_SNDTIMEO](https://linux.die.net/man/7/socket)。
不過這邊可以注意的是，因為這邊底層是走 linux socket 的方式，所以是走上述的方法去設定，若今天改走 RDMA 的話，作法就會完全不同。

``` c
0696     connection->last_received = jiffies;
0697 
0698     rcu_read_lock();
0699     nc = rcu_dereference(connection->transport.net_conf);
0700     ping_timeo = nc->ping_timeo;
0701     ping_int = nc->ping_int;
0702     rcu_read_unlock();
0703 
0704     /* Make sure we are "uncorked", otherwise we risk timeouts,
0705      * in case this is a reconnect and we had been corked before. */
0706     drbd_uncork(connection, CONTROL_STREAM);
0707     drbd_uncork(connection, DATA_STREAM);
0708 
0709     /* Make sure the handshake happens without interference from other threads,
0710      * or the challenge respons authentication could be garbled. */
0711     mutex_lock(&connection->mutex[DATA_STREAM]);
0712     have_mutex = true;
0713     transport->ops->set_rcvtimeo(transport, DATA_STREAM, ping_timeo * 4 * HZ/10);
0714     transport->ops->set_rcvtimeo(transport, CONTROL_STREAM, ping_int * HZ);
```

接下來嘗試去發送一些控制訊息給對面，譬如自己的DRBD版本的範圍，如下列`drbd_send_features`內所見


``` c
7332 static int drbd_send_features(struct drbd_connection *connection)
7333 {
7334     struct p_connection_features *p;
7335 
7336     p = __conn_prepare_command(connection, sizeof(*p), DATA_STREAM);
7337     if (!p)
7338         return -EIO;
7339     memset(p, 0, sizeof(*p));
7340     p->protocol_min = cpu_to_be32(PRO_VERSION_MIN);
7341     p->protocol_max = cpu_to_be32(PRO_VERSION_MAX);
7342     p->sender_node_id = cpu_to_be32(connection->resource->res_opts.node_id);
7343     p->receiver_node_id = cpu_to_be32(connection->peer_node_id);
7344     p->feature_flags = cpu_to_be32(PRO_FEATURES);
7345     return __send_command(connection, -1, P_CONNECTION_FEATURES, DATA_STREAM);
7346 }
```
``` c
0716     h = drbd_do_features(connection);
0717     if (h < 0)
0718         goto abort;
0719     if (h == 0)
0720         goto retry;
```
中間又重新設定了一下 receive 的 timeout，而且只有針對 **DATA_STREAM**，意義不明。
最後呼叫 `__drbd_send_protocol` 將一些 **net_conf** 內的資料送過去。
``` c
0732 
0733     transport->ops->set_rcvtimeo(transport, DATA_STREAM, MAX_SCHEDULE_TIMEOUT);
0734 
0735     discard_my_data = test_bit(CONN_DISCARD_MY_DATA, &connection->flags);
0736 
0737     if (__drbd_send_protocol(connection, P_PROTOCOL) == -EOPNOTSUPP)
0738         goto abort;
```
最後面這段還不是很清楚在幹什麼，必須要更深入的理解細節，才能瞭解為什麼這邊又要跑一個 worker，裡面又會呼叫到 **conn_connect2** 來處理。

``` c
0767     if (connection->agreed_pro_version >= 110) {
0768         if (resource->res_opts.node_id < connection->peer_node_id) {
0769             kref_get(&connection->kref);
0770             kref_debug_get(&connection->kref_debug, 11);
0771             connection->connect_timer_work.cb = connect_work;
0772             timeout = twopc_retry_timeout(resource, 0);
0773             drbd_debug(connection, "Waiting for %ums to avoid transaction "
0774                    "conflicts\n", jiffies_to_msecs(timeout));
0775             connection->connect_timer.expires = jiffies + timeout;
0776             add_timer(&connection->connect_timer);
0777         }
0778     }
```


Summary
=======
- Kernel 這邊有非常多的 thread 在運行，同時還有很複雜的 state 狀態跑來跑去，要完整瞭解整個架構以及運作邏輯需要不少時間去測試。
- 目前網路上幾乎沒有這方面的文件，就連官方網站也沒有文章說明底層的架構，這部分都只能依靠上層的應用說法與程式碼自己拼湊出這一切。
- 整個 Coonection 內還包含了 **DATA_STREAM**與 **DATA_STREAM**，這部分的用途差異，實際上怎運過還必須要在更仔細地觀看相關函式以及 **transport class TCP** 底層的實作才有機會瞭解。

