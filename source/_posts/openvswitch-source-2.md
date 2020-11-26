---
title: OpenvSwitch - 2
date: '2013-12-09 10:03'
comments: true
tags:
  - SDN
  - Network
  - OpenvSwitch
  - SourceCode
keywords: 'SDN,OpenvSwitch,OVS,Kernel'
abbrlink: 16391
---
ovs-vsctl add-port br eth1
(netlink)


kernel side:

- Register a generic netlink
- Call function when receive command by netlink from userspace (ovs-vsctl)
- Use the interface name to get the net_device
- Register the send and receive event handler.

**Register generic netlink**
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
- 從netlink收到 **OVS_VPORT_CMD_NEW**的cmd時，就會執行 **ovs_vport_cmd_new**

``` c
static int ovs_vport_cmd_new(struct sk_buff *skb, struct genl_info *info)
{
        struct nlattr **a = info->attrs;
        struct ovs_header *ovs_header = info->userhdr;
        struct vport_parms parms;
        struct sk_buff *reply;
        struct vport *vport;
        struct datapath *dp;
        u32 port_no;
        int err;

        err = -EINVAL;
        if (!a[OVS_VPORT_ATTR_NAME] || !a[OVS_VPORT_ATTR_TYPE] ||
            !a[OVS_VPORT_ATTR_UPCALL_PID])
                goto exit;

        ovs_lock();
        dp = get_dp(sock_net(skb->sk), ovs_header->dp_ifindex);
        err = -ENODEV;
        if (!dp)
                goto exit_unlock;

        if (a[OVS_VPORT_ATTR_PORT_NO]) {
                port_no = nla_get_u32(a[OVS_VPORT_ATTR_PORT_NO]);

                err = -EFBIG;
                if (port_no >= DP_MAX_PORTS)
                        goto exit_unlock;

                vport = ovs_vport_ovsl(dp, port_no);
                err = -EBUSY;
                if (vport)
                        goto exit_unlock;
        } else {
                for (port_no = 1; ; port_no++) {
                        if (port_no >= DP_MAX_PORTS) {
                                err = -EFBIG;
                                goto exit_unlock;
                        }
                        vport = ovs_vport_ovsl(dp, port_no);
                        if (!vport)
                                break;
                }
        }

        parms.name = nla_data(a[OVS_VPORT_ATTR_NAME]);
        parms.type = nla_get_u32(a[OVS_VPORT_ATTR_TYPE]);
        parms.options = a[OVS_VPORT_ATTR_OPTIONS];
        parms.dp = dp;
        parms.port_no = port_no;
        parms.upcall_portid = nla_get_u32(a[OVS_VPORT_ATTR_UPCALL_PID]);

        vport = new_vport(&parms);
        err = PTR_ERR(vport);
        if (IS_ERR(vport))
                goto exit_unlock;

        err = 0;
        if (a[OVS_VPORT_ATTR_STATS])
                ovs_vport_set_stats(vport, nla_data(a[OVS_VPORT_ATTR_STATS]));

        reply = ovs_vport_cmd_build_info(vport, info->snd_portid, info->snd_seq,
                                         OVS_VPORT_CMD_NEW);
        if (IS_ERR(reply)) {
                err = PTR_ERR(reply);
                ovs_dp_detach_port(vport);
                goto exit_unlock;
        }

        ovs_notify(reply, info, &ovs_dp_vport_multicast_group);

exit_unlock:
        ovs_unlock();
exit:
        return err;
}

```
- 這邊會呼叫 **new_vport**

``` c
/* Called with ovs_mutex. */
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

- 這邊會呼叫 **ovs_vport_add**


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

- 這邊會執行 **vport_ops_list[i]->create(parms)**

``` c
/* List of statically compiled vport implementations.  Don't forget to also
 * add yours to the list at the bottom of vport.h. */
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

- 假設是個netdev的port,就會執行 **ovs_netdev_vport_ops**裡面的create

``` c

const struct vport_ops ovs_netdev_vport_ops = {
        .type           = OVS_VPORT_TYPE_NETDEV,
        .create         = netdev_create,
        .destroy        = netdev_destroy,
        .get_name       = ovs_netdev_get_name,
        .send           = netdev_send,
};

```

- 這時候就會執行 **netdev_create**

``` c
static struct vport *netdev_create(const struct vport_parms *parms)
{
        struct vport *vport;
        struct netdev_vport *netdev_vport;
        int err;

        vport = ovs_vport_alloc(sizeof(struct netdev_vport),
                                &ovs_netdev_vport_ops, parms);
        if (IS_ERR(vport)) {
                err = PTR_ERR(vport);
                goto error;
        }

        netdev_vport = netdev_vport_priv(vport);

        netdev_vport->dev = dev_get_by_name(ovs_dp_get_net(vport->dp), parms->name);
        if (!netdev_vport->dev) {
                err = -ENODEV;
                goto error_free_vport;
        }

        if (netdev_vport->dev->flags & IFF_LOOPBACK ||
            netdev_vport->dev->type != ARPHRD_ETHER ||
            ovs_is_internal_dev(netdev_vport->dev)) {
                err = -EINVAL;
                goto error_put;
        }

        rtnl_lock();
        err = netdev_master_upper_dev_link(netdev_vport->dev,
                                           get_dpdev(vport->dp));
        if (err)
                goto error_unlock;

        err = netdev_rx_handler_register(netdev_vport->dev, netdev_frame_hook,
                                         vport);
        if (err)
                goto error_master_upper_dev_unlink;

        dev_set_promiscuity(netdev_vport->dev, 1);
        netdev_vport->dev->priv_flags |= IFF_OVS_DATAPATH;
        rtnl_unlock();

        netdev_init();
        return vport;

error_master_upper_dev_unlink:
        netdev_upper_dev_unlink(netdev_vport->dev, get_dpdev(vport->dp));
error_unlock:
        rtnl_unlock();
error_put:
        dev_put(netdev_vport->dev);
error_free_vport:
        ovs_vport_free(vport);
error:
        return ERR_PTR(err);
}


```

- 這邊會調用 ** dev_get_by_name ** 用該name取得對應的device.
- 調用 ** netdev_rx_handler_register ** 註冊 rx handler.
-

userspace side:

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4N7G

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