---
title: DRBD v9.0 Network Work Flow(i)
date: 2017-05-11 11:44:12
tags:
	- System
	- DRBD
	- Network
keywords: DRBD
---
Architecture
============
整個 DRBD 分成 kernel space 跟 user space。
主要的功能都在 kernal space，在 9.0 版本中包含了兩個 kernel module(商業版還在多一個RDMA)，其中 **drbd_transport_tcp.ko**專心負責 TCP 連線的部分，剩下的功能都在 **drbd.ko** 中實現。
而 user space 則是負責提供用戶一個方便的操作，如`drbdadm`這方面的工具，當用戶執行如 **drbdadm up r0** 這些指令後，最後會透過 **drbdsetup** 將必要的資訊透過 **netlink**的方式送往 kernel space，而先前的 kernel module 則會在 insert 時就註冊許多 netlink event，每種type都有對應的 handler來處理。
如下圖呈現

![img](http://i.imgur.com/SNMiWYj.jpg)
<!--more-->

Environment
===========
本文使用的程式碼基於下列兩個專案，分別是
- kernel space 的 project [drbd-9.0](http://git.drbd.org/drbd-9.0.git)
- user space tool 的 project [drbd-utils](http://git.drbd.org/drbd-utils.git)

configuration
=============
本文使用以下的設定檔，並且著重於當 DRBD 啟動後，整體的網路部分是如何處理的。
```c
resource r0 {
  on hw1 {
    device /dev/drbd0;
    disk /dev/sdb1;
    address 10.11.56.2:7788;
    meta-disk internal;
  }
  on hw2 {
    device /dev/drbd0;
    disk /dev/sdb1;
    address 10.11.56.3:7788;
    meta-disk internal;
  }
}
```


Steps
=====
我們使用 `drbdadm up r0` 將特定的 resource **r0** 給叫起來，這個步驟實際上會牽扯到很多行為，本文主要研究 **user space**的行為。

### post_parse

首先，當 `drbdadm` 這隻程式起來後，內部會先執行 `post_parse` 對設定檔進行一番解析，並且將解析到的資料給存到一個 **d_resource** 的物件中
```c
1063 void post_parse(struct resources *resources, enum pp_flags flags)
1064 {
1065     struct d_resource *res;
1066     struct connection *con;
..................
1102     for_each_resource(res, resources) {
1103         struct d_host_info *host;
1104         struct mesh *mesh;
1105 
1106         if (!(flags & DRBDSETUP_SHOW)) {
1107             for_each_connection(con, &res->connections)
1108                 must_have_two_hosts(res, con);
1109         }
1110 
1111         /* Other steps make no sense. */
1112         if (!config_valid)
1113             continue;
1114 
1115         STAILQ_FOREACH(mesh, &res->meshes, link)
1116             create_connections_from_mesh(res, mesh);
1117         create_implicit_connections(res);
1118         for_each_connection(con, &res->connections)
1119             set_host_info_in_host_address_pairs(res, con);
1120         for_each_host(host, &res->all_hosts) {
1121             if (!host->node_id)
1122                 derror(host, res, "node-id");
1123         }
1124     }
```
值得注意是的後半部分網路部分的處理，目前 **drbd.conf** 支援的網路設定大致上有
- Host 直接設定
- Connection 指名哪兩台 host 要互連
- Mesh 直接用參數內的 Host 創建一個 mesh 網路

所以從 **1102** 行開始，就針對每個 resource 的網路狀態去處理，最下面的部分主要分成四個部分去看
- 如果有設定 Mesh 網路，則透過 `create_connections_from_mesh`去創建所有的 connection
- 接下來會根據設定檔去創建一個隱性的連線，我們的設定檔主要是依賴此 function 去運作的，因為我們沒有特別設定 **connection** 以及 **mesh**，所以會透過此 `create_implicit_connection` 去創建一條 connection 來使用
- 接下來從所有的 connection 物件中(可能是手動設定，也有可能是上述創建的)，去設定相關的連線地址
- 最後檢查是否所有的 host 都已經有 **node_id** 這個欄位，由於我們的設定檔也沒有寫 **node_id**，這個數值會在上述的 `set_host_info_in_host_address_pairs` 中去創立。

### create_implicit_connection
接下來看一下`create_implicit_connection`怎麼處理的
``` c
0729 static void create_implicit_connections(struct d_resource *res)
0730 {
0731     struct connection *conn;
0732     struct path *path;
0733     struct hname_address *ha;
0734     struct d_host_info *host_info;
0735     int hosts = 0;
0736 
0737     if (!STAILQ_EMPTY(&res->connections))
0738         return;
```
從這邊可以觀察到，如果你的**drbd.conf**中有使用到 **connection** 欄位的話，那這邊就直接返回，不需要幫忙產生任何 connection 使用
``` c
0740     conn = alloc_connection();
0741     conn->implicit = 1;
0742     path = alloc_path();
0743     path->implicit = 1;
0744     insert_tail(&conn->paths, path);
```
初始化相關成員，主要是 **struct connection** 以及 **struct path**，之後會再分析這些結構彼此的關係，這邊只要先知道每個 **connection**底下都會有一串 **path** 即可。
``` c
0746     for_each_host(host_info, &res->all_hosts) {
0747         if (++hosts == 3) {
0748             err("Resource %s:\n\t"
0749                 "Use explicit 'connection' sections with more than two 'on' sections.\n",
0750                     res->name);
0751             break;
0752         }
0753         if (host_info->address.af && host_info->address.addr && host_info->address.port) {
0754             ha = alloc_hname_address();
0755             ha->host_info = host_info;
0756             ha->proxy = host_info->proxy_compat_only;
0757             if (!host_info->lower) {
0758                 ha->name = STAILQ_FIRST(&host_info->on_hosts)->name;
0759             } else {
0760                 ha->name = strdup(names_to_str_c(&host_info->on_hosts, '_'));
0761                 ha->address = host_info->address;
0762                 ha->faked_hostname = 1;
0763                 ha->parsed_address = 1; /* not true, but makes dump nicer */
0764             }
0765             STAILQ_INSERT_TAIL(&path->hname_address_pairs, ha, link);
0766         }
0767     }
0768 
0769     if (hosts == 2)
0770         STAILQ_INSERT_TAIL(&res->connections, conn, link);
0771     else
0772         free_connection(conn);
```
這邊要開始針對 **host**創立對應的 connection 結構，這邊要注意的是，若 **drbd.conf**中該 host 數量是兩台以上的時候，這時候 connection 就沒有那麼簡單的去決定要使用那些 host，所以在這邊就會跳過這個情形。
針對每個 host 將其資訊都收集起來放在 **struct hname_address \*ha** 內，最後再放到 **path**裡面
所以到這個階段，一條 connection 底下有一個 path，而 path 底下有一個 list，裡面放了兩個 **ha**。
最後呼叫 MARCO **STAILQ_INSERT_TAIL** 將當前創好的 **connection** 給放到整個 **resource** 物件之中，這邊也是透過 **linklist** 的方式給綁進去。

### set_host_info_in_host_address_pairs
在 `post_parse` 中，當我們都準備好 connection 後，接下來會透過 `set_host_info_in_host_address_pairs `要處理一些 host 相關的資訊，如 **node_id**。
可以從程式碼內看到，會掃過所有的 path，然後對所有的 path 再進行一次 `_set_host_info_in_host_address_pairs` 的呼叫，在本文的範例中，因為 PATH 只有一條，所以只會被呼叫一次。
``` c
0255 static void set_host_info_in_host_address_pairs(struct d_resource *res, struct connection *conn)
0256 {
0257     struct path *path;
0258 
0259     for_each_path(path, &conn->paths)
0260         _set_host_info_in_host_address_pairs(res, conn, path);
0261 }
```
### _set_host_info_in_host_address_pairs
這邊的程式碼比較長，主要針對跟本文範例相關的邏輯為主
首先先掃過該 path 底下的**hname_address**，在之前的過程中，我們塞了兩個 **struct hname_address**進去，所以理論上這個 for 迴圈只會跑兩次而已。
由於先前創立 **hname_address** 的時候，也順便將其底下的 **host_info** 也準備好了，所以可以看到第一個 **if** 判斷旁邊也有相對應的註解。
這邊最主要的是使用 crc32 計算 address 的 hash使用，供後續產生 node_id 使用，同時把這些 host_info 都存起來

```c
0141 static void _set_host_info_in_host_address_pairs(struct d_resource *res,
0142                          struct connection *conn,
0143                          struct path *path)
0144 {
0145     struct hname_address *ha;
0146     struct d_host_info *host_info;
0147     int addr_hash[2], i = 0;
0148     struct d_host_info *host_info_array[2];
0149 
0150     STAILQ_FOREACH(ha, &path->hname_address_pairs, link) {
0151         if (ha->host_info) { /* Implicit connection have that already set. */
0152             host_info = ha->host_info;
0153             if (i == 2) {
0154                 err("LOGIC BUG in set_host_info_in_host_address_pairs()\n");
0155                 exit(20);
0156             }
0157             if (!host_info->address.addr) {
0158                 err("\"connection-mesh\" (for \"%s\") with a host (\"%s\") "
0159                         "that has no \"address\" defined\n",
0160                         res->name, ha->name);
0161                 config_valid = 0;
0162                 return;
0163             }
0164             addr_hash[i] = crc32c(0x1a656f21,
0165                     (void *)host_info->address.addr,
0166                     strlen(host_info->address.addr));
0167             host_info_array[i++] = host_info;
...........
```

最後，若這條 connection 是透過 `create_implicit_connection` 產生的，則要對 connection 兩端的 host 去產生一個 node_id 來存放，這邊使用了 `generate_implicit_node_id` 來產生 node id，若剛好兩個 hash 都一樣的話，就會發生失敗，註解中有提到失敗的原因有可能兩個 host 採用了 proxy 的架構，所以 ip address 都會相同。這種情況下就重新透過`crc32c`搭配proxy的變數來重新計算一次node id。
``` c
0224     if (conn->implicit && i == 2 && !host_info_array[0]->node_id && !host_info_array[1]->node_id) {
0225         /* This is drbd-8.3 / drbd-8.4 compatibility, auto created node-id */
0226         bool have_node_ids;
0227 
0228         have_node_ids = generate_implicit_node_id(addr_hash, host_info_array);
0229 
0230         if (!have_node_ids) {
0231             /* That might be a config with equal node addresses, since it is
0232               127.0.0.1:xxx with a proxy... */
0233             i = 0;
0234             path = STAILQ_FIRST(&conn->paths); /* there may only be one */
0235             STAILQ_FOREACH(ha, &path->hname_address_pairs, link) {
0236                 if (!ha->host_info)
0237                     continue;
0238 
0239                 if (!ha->proxy)
0240                     break;
0241 
0242                 addr_hash[i++] = crc32c(0x1a656f21,
0243                             (void *)ha->proxy->outside.addr,
0244                             strlen(ha->proxy->outside.addr));
0245             }
0246             have_node_ids = generate_implicit_node_id(addr_hash, host_info_array);
0247         }
0248         if (!have_node_ids) {
0249             err("BAD LUCK, equal hashes\n");
0250             exit(20);
0251         }
0252     }
```

當整個設定檔都解析完畢後，接下來就要處理真正的參數**up r0**了，根據下列程式碼
``` c
0326 /*  */ struct adm_cmd disconnect_cmd = {"disconnect", adm_drbdsetup, &disconnect_cmd_ctx, ACF1_DISCONNECT};
0327 static struct adm_cmd up_cmd = {"up", adm_up, ACF1_RESNAME };
0328 /*  */ struct adm_cmd res_options_cmd = {"resource-options", adm_resource, &resource_options_ctx, ACF1_RESNAME};
```
可以清楚的看到，當第二個參數是 **up** 時，實際上會呼叫 `adm_up` 來進行後續的處理。
接下來看 `adm_up` 的介紹

### adm_up
```c++
1974 /* The "main" loop iterates over resources.
1975  * This "sorts" the drbdsetup commands to bring those up
1976  * so we will later first create all objects,
1977  * then attach all local disks,
1978  * adjust various settings,
1979  * and then configure the network part */
1980 static int adm_up(const struct cfg_ctx *ctx)
1981 {
     .........
2021     return 0;
2022 }
```
可以觀察到，這隻 function 負責超多事情，基本上就是幫你把 object/disk/network 都處理完畢。這邊我們專注於 Network 相關的處理。
首先先呼叫 `set_peer_in_resource` 進行處理
``` c
1988     set_peer_in_resource(ctx->res, true);
```
### set_peer_in_resource
這邊會先掃過所有的 connection，然後對於每條connection，透過 `set_peer_in_connection` 去設定每條 connection 的 peer，同時也設定 connection 底下 path 的 peer address。

以本文的範例來說，該 resource **r0** 裡面包含兩台 host，分別是 **hw1** 以及 **hw2**。
一開始兩台 host 都必須要執行 **drbdadm** 來初始相關的功能，假設今天是 **hw1** 這台在執行。則對 hw1 來說，他看到 connection 的 peer 就要指向 **hw2**，反之亦然， hw2 所看到的 connection->peer 應該要指向 **h1** 才對。

``` c
0473 void set_peer_in_resource(struct d_resource* res, int peer_required)
0474 {
0475     struct connection *conn;
0476     int peers_addrs_set = 1;
0477 
0478     for_each_connection(conn, &res->connections) {
0479         struct path *path;
0480         set_peer_in_connection(res, conn, peer_required);
0481 
0482         for_each_path(path, &conn->paths) {
0483             if (!path->peer_address)
0484                 peers_addrs_set = 0;
0485         }
0486         create_implicit_net_options(conn);
0487     }
0488     res->peers_addrs_set = peers_addrs_set;
0489 
0490     if (!(peer_required & DRBDSETUP_SHOW))
0491         add_no_bitmap_opt(res);
0492 }
```
在設定完畢 **peer** 後，透過 `create_implicit_net_options` 去設定 network options 中的 **_name** 這個欄位而已。 
最後用一個變數**peer_addr_set**來記住當前 resource 是否已經有設定過 peer 的 address了，因為有些 command 本身不需要 peer 的參與，所以會使用這個變數來作為一些邏輯的判斷。

最後來到了整個 `adm_up` 函式的重頭戲, 在一切資訊都準備完畢後，接下來要開始在兩端 host *h1*, *h2* 建立起連線，這邊透過 **schedule_deferred_cmd** 的方式去執行三個指令，分別是 `new-peer`, `new-path` 以及 `connect`，稍後這些指令都會透過 netlink 的方式送到 kernel space 去進行真正的連線操作。
``` c
1989     for_each_connection(conn, &ctx->res->connections) {
1990         struct peer_device *peer_device;
1991 
1992         if (conn->ignore)
1993             continue;
1994 
1995         tmp_ctx.conn = conn;
1996 
1997         schedule_deferred_cmd(&new_peer_cmd, &tmp_ctx, CFG_NET_PREP_UP);
1998         schedule_deferred_cmd(&new_path_cmd, &tmp_ctx, CFG_NET_PATH);
1999         schedule_deferred_cmd(&connect_cmd, &tmp_ctx, CFG_NET_CONNECT);
2000 
2001         STAILQ_FOREACH(peer_device, &conn->peer_devices, connection_link) {
2002             struct cfg_ctx tmp2_ctx;
2003 
2004             if (STAILQ_EMPTY(&peer_device->pd_options))
2005                 continue;
2006 
2007             tmp2_ctx = tmp_ctx;
2008             tmp2_ctx.vol = peer_device->volume;
2009             schedule_deferred_cmd(&peer_device_options_cmd, &tmp2_ctx, CFG_PEER_DEVICE);
2010         }
2011     }
```

接下來看看 `new-peer`, `new-path` 以及 `connection` 實際上又做了些什麼事情。
在實際看這些指令做的事情以前，先來看看 `schedule_deferred_cmd` 怎麼處理這些指令的。
### schedule_deferred_cmd
此 function 主要是將相關的參數都收集起來放到 **struct cfg_ctx** 裡面，然後將這個要執行的指令透過 `STAILQ_INSERT_TAIL` 都方式放到一個全域的 Queue **deferred_cmds** 內。
``` c
0547 void schedule_deferred_cmd(struct adm_cmd *cmd,
0548                const struct cfg_ctx *ctx,
0549                enum drbd_cfg_stage stage)
0550 {
0551     struct deferred_cmd *d;
0552 
0553     if (stage & SCHEDULE_ONCE) {
0554         stage &= ~SCHEDULE_ONCE;
0555 
0556         STAILQ_FOREACH(d, &deferred_cmds[stage], link) {
0557             if (d->ctx.cmd == cmd &&
0558                 d->ctx.res == ctx->res &&
0559                 d->ctx.conn == ctx->conn &&
0560                 d->ctx.vol == ctx->vol)
0561                 return;
0562         }
0563     }
0564 
0565     d = calloc(1, sizeof(struct deferred_cmd));
0566     if (d == NULL) {
0567         perror("calloc");
0568         exit(E_EXEC_ERROR);
0569     }
0570 
0571     d->ctx = *ctx;
0572     d->ctx.cmd = cmd;
0573 
0574     STAILQ_INSERT_TAIL(&deferred_cmds[stage], d, link);
0575 }
```
整個程式的最後面則是會依賴 `_run_deferred_cmds` 將 queue 內的指令一個一個取出，然後透過 `__call_cmd_fn` 開始執行

### _run_deferred_cmds
``` c
0698 int _run_deferred_cmds(enum drbd_cfg_stage stage)
0699 {
0700     struct d_resource *last_res = NULL;
0701     struct deferred_cmd *d = STAILQ_FIRST(&deferred_cmds[stage]);
0702     struct deferred_cmd *t;
0703     int r;
0704     int rv = 0;
0705 
0706     if (d && adjust_with_progress) {
0707         printf("\n%15s:", drbd_cfg_stage_string[stage]);
0708         fflush(stdout);
0709     }
0710 
0711     while (d) {
0712         if (d->ctx.res->skip_further_deferred_command) {
0713             if (adjust_with_progress) {
0714                 if (d->ctx.res != last_res)
0715                     printf(" [skipped:%s]", d->ctx.res->name);
0716             } else
0717                 err("%s: %s %s: skipped due to earlier error\n",
0718                     progname, d->ctx.cmd->name, d->ctx.res->name);
0719             r = 0;
0720         } else {
0721             if (adjust_with_progress) {
0722                 if (d->ctx.res != last_res)
0723                     printf(" %s", d->ctx.res->name);
0724             }
0725             r = __call_cmd_fn(&d->ctx, KEEP_RUNNING);
0726             if (r) {
...
```

這邊可以注意的是 **iterate_path** 這個變數，如果這個變數為真的則，則該指令會針對 connection內所有的 paths 都進行一次， connection 則是在當初在 **adm_up** 時就會先透過 **tmp_ctx.conn = conn** 放進去。不過由於本文的設定檔只有一條 connection，且該 connection 上只有一個 path，所以這邊實際上也只會呼叫一次。
### __call_cmd_fn
``` c
0578 static int __call_cmd_fn(const struct cfg_ctx *ctx, enum on_error on_error)
0579 {
0580     struct d_volume *vol = ctx->vol;
0581     bool iterate_paths;
0582     int rv = 0;
0583 
0584     iterate_paths = ctx->path ? 0 : ctx->cmd->iterate_paths;
0585 
0586     if (ctx->cmd->disk_required &&
0587         (!vol->disk || !vol->meta_disk || !vol->meta_index)) {
0588         rv = 10;
0589         err("The %s command requires a local disk, but the configuration gives none.\n",
0590             ctx->cmd->name);
0591         if (on_error == EXIT_ON_FAIL)
0592             exit(rv);
0593         return rv;
0594     }
0595 
0596     if (iterate_paths) {
0597         struct cfg_ctx tmp_ctx = *ctx;
0598         struct path *path;
0599 
0600         for_each_path(path, &tmp_ctx.conn->paths) {
0601             tmp_ctx.path = path;
0602             rv = tmp_ctx.cmd->function(&tmp_ctx);
0603             if (rv >= 20) {
0604                 if (on_error == EXIT_ON_FAIL)
0605                     exit(rv);
0606             }
0607 
0608         }
0609     } else {
0610         rv = ctx->cmd->function(ctx);
0611         if (rv >= 20) {
0612             if (on_error == EXIT_ON_FAIL)
0613                 exit(rv);
0614         }
0615     }
0616     return rv;
0617 }
```
最後要來看這些指令怎麼往下運行的，不論是 **new-peer**, **new-path** 或是 **connect**，其實最後都是依靠 `drbdsetup` 這隻程式在來運行，所以這邊基本上都是收集好參數後透過 `system` 的方式將該指令叫起來去執行。
``` c
1705 static int adm_connect(const struct cfg_ctx *ctx)
1706 {
1707     struct d_resource *res = ctx->res;
1708     struct connection *conn = ctx->conn;
1709     char *argv[MAX_ARGS];
1710     int argc = 0;
1711 
1712     argv[NA(argc)] = drbdsetup;
1713     argv[NA(argc)] = (char *)ctx->cmd->name; /* "connect" */
1714     argv[NA(argc)] = ssprintf("%s", res->name);
1715     argv[NA(argc)] = ssprintf("%s", conn->peer->node_id);
1716 
1717     add_setup_options(argv, &argc, ctx->cmd->drbdsetup_ctx);
1718     argv[NA(argc)] = 0;
1719 
1720     return m_system_ex(argv, SLEEPS_SHORT, res->name);
1721 }
1722 
1723 static int adm_new_peer(const struct cfg_ctx *ctx)
1724 {
1725     struct d_resource *res = ctx->res;
1726     struct connection *conn = ctx->conn;
1727 
1728     char *argv[MAX_ARGS];
1729     int argc = 0;
1730 
1731     bool reset = (ctx->cmd == &net_options_defaults_cmd);
1732 
1733     argv[NA(argc)] = drbdsetup;
1734     argv[NA(argc)] = (char *)ctx->cmd->name; /* "new-peer", "net-options" */
1735     argv[NA(argc)] = ssprintf("%s", res->name);
1736     argv[NA(argc)] = ssprintf("%s", conn->peer->node_id);
1737 
1738     if (reset)
1739         argv[NA(argc)] = "--set-defaults";
1740 
1741     if (!strncmp(ctx->cmd->name, "net-options", 11))
1742         del_opt(&conn->net_options, "transport");
1743 
1744     make_options(argv[NA(argc)], &conn->net_options);
1745 
1746     add_setup_options(argv, &argc, ctx->cmd->drbdsetup_ctx);
1747     argv[NA(argc)] = 0;
1748 
1749     return m_system_ex(argv, SLEEPS_SHORT, res->name);
1750 }
1751 
1752 static int adm_path(const struct cfg_ctx *ctx)
1753 {
1754     struct d_resource *res = ctx->res;
1755     struct connection *conn = ctx->conn;
1756     struct path *path = ctx->path;
1757 
1758     char *argv[MAX_ARGS];
1759     int argc = 0;
1760 
1761     argv[NA(argc)] = drbdsetup;
1762     argv[NA(argc)] = (char *)ctx->cmd->name; /* add-path, del-path */
1763     argv[NA(argc)] = ssprintf("%s", res->name);
1764     argv[NA(argc)] = ssprintf("%s", conn->peer->node_id);
1765 
1766     argv[NA(argc)] = ssprintf_addr(path->my_address);
1767     argv[NA(argc)] = ssprintf_addr(path->connect_to);
1768 
1769     add_setup_options(argv, &argc, ctx->cmd->drbdsetup_ctx);
1770     argv[NA(argc)] = 0;
1771 
1772     return m_system_ex(argv, SLEEPS_SHORT, res->name);
1773 }
```
在 `drbdsetup` 中，可以看到關於這三個指令對應的資訊，這些指令的原型是 **struct drbd_cmd**，當 **drbdsetup** 被呼叫後，對應的指令就會跑到對應的 **drbd_cmd**中去執行，最後都會執行到 **drbd_cmd** 裡面的 **function (fptr)**來處理。
``` c
0229 struct drbd_cmd {
0230     const char* cmd;
0231     enum cfg_ctx_key ctx_key;
0232     int cmd_id;
0233     int tla_id; /* top level attribute id */
0234     int (*function)(struct drbd_cmd *, int, char **);
0235     struct drbd_argument *drbd_args;
0236     int (*show_function)(struct drbd_cmd*, struct genl_info *, void *u_ptr);
0237     struct option *options;
0238     bool missing_ok;
0239     bool warn_on_missing;
0240     bool continuous_poll;
0241     bool set_defaults;
0242     bool lockless;
0243     struct context_def *ctx;
0244     const char *summary;
0245 };
```
``` c
0397     {"connect", CTX_PEER_NODE,
0398         DRBD_ADM_CONNECT, DRBD_NLA_CONNECT_PARMS,
0399         F_CONFIG_CMD,
0400      .ctx = &connect_cmd_ctx,
0401      .summary = "Attempt to (re)establish a replication link to a peer host." },
```
``` c
0403     {"new-peer", CTX_PEER_NODE,
0404         DRBD_ADM_NEW_PEER, DRBD_NLA_NET_CONF,
0405         F_CONFIG_CMD,
0406      .ctx = &new_peer_cmd_ctx,
0407      .summary = "Make a peer host known to a resource." },
```
``` c
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
在三個 case 中，三個指令對應的 function 其實都指向了 **generic_config_cmd** 這 function，而 **generic_config_cmd** 則再繼續呼叫 **_generic_config_cmd** 繼續往下處理

### _generic_config_cmd
到這一步後，就是根據先前指令中的一些資訊，組出對應的 **netlink header**，最後透過
`genl_send` 將該命令透過 **netlink** 送到 kernel 去，然後再使用 `genl_recv_msgs` 的方式接收回來的訊息，確認事情完成後就結束。

```c
1136 static int _generic_config_cmd(struct drbd_cmd *cm, int argc, char **argv)
1137 {
1138     struct drbd_argument *ad;
1139     struct nlattr *nla;
1140     struct option *options;
1141     int c, i;
1142     int rv;
1143     char *desc = NULL; /* error description from kernel reply message */
1144 
1145     struct drbd_genlmsghdr *dhdr;
1146     struct msg_buff *smsg;
1147     struct iovec iov;
1148     struct nlmsghdr *nlh;
1149     struct drbd_genlmsghdr *dh;
1150     struct timespec retry_timeout = {
1151         .tv_nsec = 62500000L,  /* 1/16 second */
1152     };
...
```


Summary
=======
- 本文到這邊目前已經大致瞭解 user space 的流程，接下來要探討 **new-peer**, **new-path** 以及 **connect** 這三個指令在 kernel 中的流程。
