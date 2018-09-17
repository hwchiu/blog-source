---
layout: post
title: OpenvSwitch source code(2)
date: '2013-12-13 09:18'
comments: true
tags:
  - SDN
  - Network
  - OpenvSwitch
  - SourceCode
keywords: 'SDN,OpenvSwitch,OVS,Kernel'
abbrlink: 27903
---
##Brief###
當使用 **ovs-vsctl add-port br0 eth1**的時候，實際上會做什麼事情，這邊分成兩個層面來看

- User space
- Kernel space

而這之中則會透過 **generic netlink**來溝通

<!--more-->

#### Uer space####

這部分還有點卡住，對 **ovsdb** 有更瞭解後再補充
bridge_reconfigure
bridge_refresh_ofp_port
bridge_refresh_one_ofp_port
iface_create
iface_do_create
ofproto_port_add


VLOG_INFO


``` c
static int
port_add(struct ofproto *ofproto_, struct netdev *netdev)
{
...
	error = dpif_port_add(ofproto->backer->dpif, netdev, &port_no);

...
}
```

``` c
int
dpif_port_add(struct dpif *dpif, struct netdev *netdev, odp_port_t *port_nop)
{
    const char *netdev_name = netdev_get_name(netdev);
    odp_port_t port_no = ODPP_NONE;
    int error;

    COVERAGE_INC(dpif_port_add);

    if (port_nop) {
        port_no = *port_nop;
    }

    error = dpif->dpif_class->port_add(dpif, netdev, &port_no);
    if (!error) {
        VLOG_DBG_RL(&dpmsg_rl, "%s: added %s as port %"PRIu32,
                    dpif_name(dpif), netdev_name, port_no);
    } else {
        VLOG_WARN_RL(&error_rl, "%s: failed to add %s as port: %s",
                     dpif_name(dpif), netdev_name, ovs_strerror(error));
        port_no = ODPP_NONE;
    }
    if (port_nop) {
        *port_nop = port_no;
    }
    return error;
}

```


``` c
static void
dp_initialize(void)
{
    static struct ovsthread_once once = OVSTHREAD_ONCE_INITIALIZER;

    if (ovsthread_once_start(&once)) {
        int i;

        for (i = 0; i < ARRAY_SIZE(base_dpif_classes); i++) {
            dp_register_provider(base_dpif_classes[i]);
        }
        ovsthread_once_done(&once);
    }
}
```

- 這邊會去註冊每個 **base_dipf_classes**.

``` c
struct dpif_class {
....
    /* Adds 'netdev' as a new port in 'dpif'.  If '*port_no' is not
     * UINT32_MAX, attempts to use that as the port's port number.
     *
     * If port is successfully added, sets '*port_no' to the new port's
     * port number.  Returns EBUSY if caller attempted to choose a port
     * number, and it was in use. */
    int (*port_add)(struct dpif *dpif, struct netdev *netdev,
                    odp_port_t *port_no);
...

}
```

- dpif_class是一個base class,裡面存放的都是function pointer.


``` c
static const struct dpif_class *base_dpif_classes[] = {
#ifdef LINUX_DATAPATH
    &dpif_linux_class,
#endif
    &dpif_netdev_class,
};

```

- 這邊會根據type去實例化不同類型的 **dpif_class**,這邊我們關心的是 **dpif_linux_class**.



``` c
const struct dpif_class dpif_linux_class = {
    "system",
    dpif_linux_enumerate,
    NULL,
    dpif_linux_open,
    dpif_linux_close,
    dpif_linux_destroy,
    NULL,                       /* run */
    NULL,                       /* wait */
    dpif_linux_get_stats,
    dpif_linux_port_add,
    dpif_linux_port_del,
    dpif_linux_port_query_by_number,
    dpif_linux_port_query_by_name,
    dpif_linux_get_max_ports,
    dpif_linux_port_get_pid,
    dpif_linux_port_dump_start,
    dpif_linux_port_dump_next,
    dpif_linux_port_dump_done,
    dpif_linux_port_poll,
    dpif_linux_port_poll_wait,
    dpif_linux_flow_get,
    dpif_linux_flow_put,
    dpif_linux_flow_del,
    dpif_linux_flow_flush,
    dpif_linux_flow_dump_start,
    dpif_linux_flow_dump_next,
    dpif_linux_flow_dump_done,
    dpif_linux_execute,
    dpif_linux_operate,
    dpif_linux_recv_set,
    dpif_linux_queue_to_priority,
    dpif_linux_recv,
    dpif_linux_recv_wait,
    dpif_linux_recv_purge,
};
```

- 這邊定義了 dpif_class的一些操作function.
- 我們關心的 **port_add** 實際上對應的是 **dpif_linux_port_add**

``` c
static int
dpif_linux_port_add(struct dpif *dpif_, struct netdev *netdev,
                    odp_port_t *port_nop)
{
    struct dpif_linux *dpif = dpif_linux_cast(dpif_);
    int error;

    ovs_mutex_lock(&dpif->upcall_lock);
    error = dpif_linux_port_add__(dpif_, netdev, port_nop);
    ovs_mutex_unlock(&dpif->upcall_lock);

    return error;
}
```

- 這邊會呼叫 **dpif_linux_port_add__**去創立vport.

``` c
static int
dpif_linux_port_add__(struct dpif *dpif_, struct netdev *netdev,
                      odp_port_t *port_nop)
{
....
    request.cmd = OVS_VPORT_CMD_NEW;
    request.dp_ifindex = dpif->dp_ifindex;
    request.type = netdev_to_ovs_vport_type(netdev);
...
	 error = dpif_linux_vport_transact(&request, &reply, &buf);
...
}
```

- 這邊會設定request的cmd為 **OVS_VPORT_CMD_NEW**
- 使用 **dpif_linux_vport_transact** 把這個request透過netlink的方式送到kernel端，kernel端會再根據這個request的cmd來執行特定的function.

#### Gereric netlink ####

``` c
static int dp_register_genl(void)
{
        int n_registered;
        int err;
        int i;

        n_registered = 0;
        for (i = 0; i < ARRAY_SIZE(dp_genl_families); i++) {
                const struct genl_family_and_ops *f = &dp_genl_families[i];

                err = genl_register_family_with_ops(f->family, f->ops,
                                                    f->n_ops);
                if (err)
                        goto error;
                n_registered++;

                if (f->group) {
                        err = genl_register_mc_group(f->family, f->group);
                        if (err)
                                goto error;
                }
        }

        return 0;

error:
        dp_unregister_genl(n_registered);
        return err;
}
```

- 當 **datapath** kernel module被載入的時候，會執行對應的 **init**,裡面會執行 **dp_register_genl**.
- 這邊會呼叫所有的 **dp_genl_families** 來註冊 **generic netlink**相關的function.


``` c
static const struct genl_family_and_ops dp_genl_families[] = {
        { &dp_datapath_genl_family,
          dp_datapath_genl_ops, ARRAY_SIZE(dp_datapath_genl_ops),
          &ovs_dp_datapath_multicast_group },
        { &dp_vport_genl_family,
          dp_vport_genl_ops, ARRAY_SIZE(dp_vport_genl_ops),
          &ovs_dp_vport_multicast_group },
        { &dp_flow_genl_family,
          dp_flow_genl_ops, ARRAY_SIZE(dp_flow_genl_ops),
          &ovs_dp_flow_multicast_group },
        { &dp_packet_genl_family,
          dp_packet_genl_ops, ARRAY_SIZE(dp_packet_genl_ops),
          NULL },
};

```

- 這邊定義了所有 **gerneric netlink**相關type的結構成員 (dp,vport,flow,packet)


``` c
static struct genl_ops dp_vport_genl_ops[] = {
        { .cmd = OVS_VPORT_CMD_NEW,
          .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
          .policy = vport_policy,
          .doit = ovs_vport_cmd_new
        },
        { .cmd = OVS_VPORT_CMD_DEL,
          .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
          .policy = vport_policy,
          .doit = ovs_vport_cmd_del
        },
        { .cmd = OVS_VPORT_CMD_GET,
          .flags = 0,               /* OK for unprivileged users. */
          .policy = vport_policy,
          .doit = ovs_vport_cmd_get,
          .dumpit = ovs_vport_cmd_dump
        },
        { .cmd = OVS_VPORT_CMD_SET,
          .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
          .policy = vport_policy,
          .doit = ovs_vport_cmd_set,
        },
};

```

- 這邊定義了 **gerneric netlink** 相關的vport operation.
- 當 cmd 是 **OVS_VPORT_CMD_NEW**的時候，就會執行對應的 function handler **ovs_vport_cmd_new**


#### Kernel space ####


``` c
static int ovs_vport_cmd_new(struct sk_buff *skb, struct genl_info *info)
{
...
        parms.name = nla_data(a[OVS_VPORT_ATTR_NAME]);
        parms.type = nla_get_u32(a[OVS_VPORT_ATTR_TYPE]);
        parms.options = a[OVS_VPORT_ATTR_OPTIONS];
        parms.dp = dp;
        parms.port_no = port_no;
        parms.upcall_portid = nla_get_u32(a[OVS_VPORT_ATTR_UPCALL_PID]);

        vport = new_vport(&parms);
...
}
```

-  struct vport_parms parms 用來記錄一些vport的資訊
-  填寫相關訊息後，就呼叫 **new_vport**來創立 vport

``` c
static struct vport *new_vport(const struct vport_parms *parms)
{
        struct vport *vport;

        vport = ovs_vport_add(parms);
        if (!IS_ERR(vport)) {
                struct datapath *dp = parms->dp;
                struct hlist_head *head = vport_hash_bucket(dp, vport->port_no);

                hlist_add_head_rcu(&vport->dp_hash_node, head);
        }
        return vport;
}
```

- 這邊會呼叫 ***ovs_vport_add** 來創立該vport

``` c
struct vport *ovs_vport_add(const struct vport_parms *parms)
{
        struct vport *vport;
        int err = 0;
        int i;

        for (i = 0; i < ARRAY_SIZE(vport_ops_list); i++) {
                if (vport_ops_list[i]->type == parms->type) {
                        struct hlist_head *bucket;

                        vport = vport_ops_list[i]->create(parms);
                        if (IS_ERR(vport)) {
                                err = PTR_ERR(vport);
                                goto out;
                        }

                        bucket = hash_bucket(ovs_dp_get_net(vport->dp),
                                             vport->ops->get_name(vport));
                        hlist_add_head_rcu(&vport->hash_node, bucket);
                        return vport;
                }
        }

        err = -EAFNOSUPPORT;

out:
        return ERR_PTR(err);
}

```

- 掃過所有 **vport_ops_list**, 如果那個ops的 **type**跟傳進來的type是一樣的，那就呼叫對應的 **create** function.

``` c
static const struct vport_ops *vport_ops_list[] = {
        &ovs_netdev_vport_ops,
        &ovs_internal_vport_ops,
#if IS_ENABLED(CONFIG_NET_IPGRE_DEMUX)
        &ovs_gre_vport_ops,
        &ovs_gre64_vport_ops,
#endif
        &ovs_vxlan_vport_ops,
        &ovs_lisp_vport_ops,
};
```

- vport_ops 有多種type.這邊我們專注於 **ovs_netdev_vport_ops**這種type.


``` c
const struct vport_ops ovs_netdev_vport_ops = {
        .type           = OVS_VPORT_TYPE_NETDEV,
        .create         = netdev_create,
        .destroy        = netdev_destroy,
        .get_name       = ovs_netdev_get_name,
        .send           = netdev_send,
};
```

- vport當type是 **netdev**的時候，成員的function pointer就會按此設定為各個function.
- 可以看到他的 **create** 對應到 **netdev_create**


``` c
static struct vport *netdev_create(const struct vport_parms *parms)
...
        err = netdev_rx_handler_register(netdev_vport->dev, netdev_frame_hook,
                                         vport);
...                                        
```

- 這邊註冊netdevice的receive handler, handler為 **netdev_frame_hook**

``` c
int netdev_rx_handler_register(struct net_device *dev,
                                rx_handler_func_t *rx_handler,
                                void *rx_handler_data)
{
         ASSERT_RTNL();
 
         if (dev->rx_handler)
                 return -EBUSY;
 
         /* Note: rx_handler_data must be set before rx_handler */
         rcu_assign_pointer(dev->rx_handler_data, rx_handler_data);
         rcu_assign_pointer(dev->rx_handler, rx_handler);
 
        return 0;
}
```

- 這邊會把設定該 **net_device**中的兩個pointer
- **rx_handler_data** 這邊就設定成該vport.
- **rx_handler** 接收到封包的處理函式，這邊就是netdev_frame_hook。
