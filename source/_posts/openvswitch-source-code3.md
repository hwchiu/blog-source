---
title: OpenvSwitch source code(3)
keywords: 'OpenvSwitch,OVS,SourceCOde,netlink,queue,implementation'
date: '2013-12-22 14:20'
comments: true
tags:
  - SDN
  - Network
  - OpenvSwitch
  - SourceCode
abbrlink: 64766
description: 這篇文章帶領大家透過閱讀原始碼的方來學習如何 OpenvSwitch 是如何處理封包的，當底層的 Kernel Switch(datapath) 沒有辦法轉發封包時，要如何將該封包送到上層的 User Space Table 進行 Openflow 規則的查詢。這部份牽扯到資料如何橫跨於 User-Space 以及 Kernel-Space.

---

# Preface
這邊要探討的是當網卡收到封包後，在 **KERNEL** 中由下往上的流程

![test.png](http://user-image.logdown.io/user/415/blog/415/post/168532/USgXG7XQxy3ZF5Qh1RY7_test.png)

<!--more-->


## netdev_frame_hook
``` c
static rx_handler_result_t netdev_frame_hook(struct sk_buff **pskb)
{
        struct sk_buff *skb = *pskb;
        struct vport *vport;

        if (unlikely(skb->pkt_type == PACKET_LOOPBACK))
                return RX_HANDLER_PASS;

        vport = ovs_netdev_get_vport(skb->dev);

        netdev_port_receive(vport, skb);

        return RX_HANDLER_CONSUMED;
}
```

<!--more-->



- **net_device**收到封包後，變變呼叫這個fucntion call來處理。
- 先判斷是不是LOOPBACK的，是的話就不需要處理了。
- 透過 **ovs_netdev_get_vport**取得該dev對應的vport
- 呼叫 **netdev_port_receive**來處理這個封包


## netdev_port_receive

``` c
static void netdev_port_receive(struct vport *vport, struct sk_buff *skb)
{
        if (unlikely(!vport))
                goto error;
        if (unlikely(skb_warn_if_lro(skb)))
                goto error;
        /* Make our own copy of the packet.  Otherwise we will mangle the
         * packet for anyone who came before us (e.g. tcpdump via AF_PACKET).
         * (No one comes after us, since we tell handle_bridge() that we took
         * the packet.) */
        skb = skb_share_check(skb, GFP_ATOMIC);
        if (unlikely(!skb))
                return;

        skb_push(skb, ETH_HLEN);
        ovs_skb_postpush_rcsum(skb, skb->data, ETH_HLEN);

        ovs_vport_receive(vport, skb, NULL);
        return;

error:
        kfree_skb(skb);
}
```

- **skb_warn_if_lro**判斷其LRO的設定有沒有問題
- **skb_push**調整skb中的data指標
- ovs_skb_postpush_rcsum 處理ip checksum。
- 呼叫 **ovs_vport_receive**繼續處理

## ovs_vport_receive

``` c
void ovs_vport_receive(struct vport *vport, struct sk_buff *skb,
                       struct ovs_key_ipv4_tunnel *tun_key)
{
        struct pcpu_tstats *stats;

        stats = this_cpu_ptr(vport->percpu_stats);
        u64_stats_update_begin(&stats->syncp);
        stats->rx_packets++;
        stats->rx_bytes += skb->len;
        u64_stats_update_end(&stats->syncp);

        OVS_CB(skb)->tun_key = tun_key;
        ovs_dp_process_received_packet(vport, skb);
}
```

- **struct pcpu_tstats** 紀錄當前cpu對於封包的一些計數 (rx,tx)
- 由 **this_cpu_ptr**取得這個vport的(tx,rx) counter.
- Update counters (packets,bytes)
- 透過 **ovs_dp_process_received_packet**繼續處理

## ovs_dp_process_received_packet

- 這個function的目的就是
	1. 由skb內取出封包各欄位的資訊(L2,L3,L4)
  2. 去查詢該 **datapath**的flow table中否有符合的flow entry
  3. 有找到，就執行對應的action.
  4. 沒有找到，就執行 **ovs_dp_upcall**送到 **user space**


``` c
void ovs_dp_process_received_packet(struct vport *p, struct sk_buff *skb)
{
        struct datapath *dp = p->dp;
        struct sw_flow *flow;
        struct dp_stats_percpu *stats;
        struct sw_flow_key key;
        u64 *stats_counter;
        int error;

        stats = this_cpu_ptr(dp->stats_percpu);

        /* Extract flow from 'skb' into 'key'. */
        error = ovs_flow_extract(skb, p->port_no, &key);
        if (unlikely(error)) {
                kfree_skb(skb);
                return;
        }

        /* Look up flow. */
        flow = ovs_flow_lookup(rcu_dereference(dp->table), &key);
        if (unlikely(!flow)) {
                struct dp_upcall_info upcall;

                upcall.cmd = OVS_PACKET_CMD_MISS;
                upcall.key = &key;
                upcall.userdata = NULL;
                upcall.portid = p->upcall_portid;
                ovs_dp_upcall(dp, skb, &upcall);
                consume_skb(skb);
                stats_counter = &stats->n_missed;
                goto out;
        }

        OVS_CB(skb)->flow = flow;
        OVS_CB(skb)->pkt_key = &key;

        stats_counter = &stats->n_hit;
        ovs_flow_used(OVS_CB(skb)->flow, skb);
        ovs_execute_actions(dp, skb);

out:
        /* Update datapath statistics. */
        u64_stats_update_begin(&stats->sync);
        (*stats_counter)++;
        u64_stats_update_end(&stats->sync);
}

```

- 先由 **vport** 取得 對應的 **datapath**
- 由 **this_cpu_ptr**取得這個datapath的(tx,rx) counter.
- 透過 **ovs_flow_extract** 把 **skb**, **vport** 的資訊都寫入 **sw_flow_key key**之中
- 呼叫 **ovs_flow_lookup** 去查詢這個packet在table之中有沒有match的flow entry.
- 如果沒有找到，那就透過 **upcall**的方式，把這個flow_miss告訴 **ovs-vswitchd**去處理
- 如果有找到，先透過 **ovs_flow_used**更新該flow的一些資訊(usedtime,packet,byte,tcp_flag),接者在呼叫 **ovs_execute_actions** 執行這個flow 對應的actions


## ovs_flow_extract
```c
int ovs_flow_extract(struct sk_buff *skb, u16 in_port, struct sw_flow_key *key)
{
        int error;
        struct ethhdr *eth;

        memset(key, 0, sizeof(*key));

        key->phy.priority = skb->priority;
        if (OVS_CB(skb)->tun_key)
                memcpy(&key->tun_key, OVS_CB(skb)->tun_key, sizeof(key->tun_key));
        key->phy.in_port = in_port;
        key->phy.skb_mark = skb->mark;

		skb_reset_mac_header(skb);

```

- OVS_CB是一個marco,會把skbuff中的cb區域拿來使用，並且轉型成 ovs_skb_cb
**#define OVS_CB(skb) ((struct ovs_skb_cb *)(skb)->cb)**
- 如果該packet有使用 **tunnel_key**的話，就把該 **tun_key**給複製到 **key**。
- 把收到封包的port number也記錄到key裡面( ingress port)
- 使用 **skb_reset_mac_header** 得到 mac header (放在 skb->mac_header)

```c
        /* Link layer.  We are guaranteed to have at least the 14 byte Ethernet
         * header in the linear data area.
         */
        eth = eth_hdr(skb);
        memcpy(key->eth.src, eth->h_source, ETH_ALEN);
        memcpy(key->eth.dst, eth->h_dest, ETH_ALEN);

        __skb_pull(skb, 2 * ETH_ALEN);
        /* We are going to push all headers that we pull, so no need to
         * update skb->csum here. */

        if (vlan_tx_tag_present(skb))
                key->eth.tci = htons(vlan_get_tci(skb));
        else if (eth->h_proto == htons(ETH_P_8021Q))
                if (unlikely(parse_vlan(skb, key)))
                        return -ENOMEM;

        key->eth.type = parse_ethertype(skb);
        if (unlikely(key->eth.type == htons(0)))
                return -ENOMEM;

        skb_reset_network_header(skb);
        __skb_push(skb, skb->data - skb_mac_header(skb));

        ....
```

- 取得 ethernet header,並且把 sourcee/destinaion mac address給複製到key。
- 透過 **__skb_pull**,把 **skb-data**給往下指 **ETH_ALEN*2**
- 檢查有沒有用 **vlan tag**,有的話就把tci給抓近來
- 使用 **parse_ethertype** 得到該封包的 ethernet type
- 使用 **skb_reset_network_header** 得到 network header (放在 skb->network_header)
- 透過 **__skb_push** 把skb中的data指標往上移(這樣可以取回mac header的一些資訊)，供未來使用
- 後面就是針對 (IP,IPV6,ARP)等在做更細部的資料取得


``` c
struct sw_flow *ovs_flow_lookup(struct flow_table *tbl,
                                const struct sw_flow_key *key)
{
        struct sw_flow *flow = NULL;
        struct sw_flow_mask *mask;

        list_for_each_entry_rcu(mask, tbl->mask_list, list) {
                flow = ovs_masked_flow_lookup(tbl, key, mask);
                if (flow)  /* Found */
                        break;
        }

        return flow;
}
```

-  從 **datapath**的**flow table**中先取得所有的 **mask_list**
-  使用 **ovs_masked_flow_lookup**去搜尋進來的封包是否有match
-  最後回傳 flow.

``` c
static struct sw_flow *ovs_masked_flow_lookup(struct flow_table *table,
                                    const struct sw_flow_key *unmasked,
                                    struct sw_flow_mask *mask)
{
        struct sw_flow *flow;
        struct hlist_head *head;
        int key_start = mask->range.start;
        int key_end = mask->range.end;
        u32 hash;
        struct sw_flow_key masked_key;

        ovs_flow_key_mask(&masked_key, unmasked, mask);
        hash = ovs_flow_hash(&masked_key, key_start, key_end);
        head = find_bucket(table, hash);
        hlist_for_each_entry_rcu(flow, head, hash_node[table->node_ver]) {
                if (flow->mask == mask &&
                    __flow_cmp_masked_key(flow, &masked_key,
                                          key_start, key_end))
                        return flow;
        }
        return NULL;
}
```

- OVS 2.0後增加對megaflow的支持，所以在kernel端也可以支援wildcard的flow matching.
- **sw_flow_key**實際上就是個wildcard，每個進來的封包都先跟wildcard做 **ovs_flow_key_mask**,然後在用mask後的結果去table中尋找有沒有可以match的
- 使用mask過後的結果來做hash,並且透過 **find_bucket**找到那個hash值所在的bucket
- 針對那個bucket中所有的flow去做比對，如果 mask相同且 **__flow_cmp__masked**結果為真，就代表match.
- key的start & end 還不是很明瞭其目的以及用途，待釐清

``` c
void ovs_flow_key_mask(struct sw_flow_key *dst, const struct sw_flow_key *src,
                       const struct sw_flow_mask *mask)
{
        const long *m = (long *)((u8 *)&mask->key + mask->range.start);
        const long *s = (long *)((u8 *)src + mask->range.start);
        long *d = (long *)((u8 *)dst + mask->range.start);
        int i;

        /* The memory outside of the 'mask->range' are not set since
         * further operations on 'dst' only uses contents within
         * 'mask->range'.
         */
        for (i = 0; i < range_n_bytes(&mask->range); i += sizeof(long))
                *d++ = *s++ & *m++;
}
```

- 把 **src**用 **mask**去處理，結果放到 **dst**上
- 這邊可以看到做mask的方法就是不停地用 &來取結果而已。

## Found
尋找到flow後
1. 更新該flow的一些統計資訊 ( **ovs_flow_used**)
2. 執行該flow entry上的actions (**ovs_execute_actions**)

### ovs_flow_used
``` c
void ovs_flow_used(struct sw_flow *flow, struct sk_buff *skb)
{
        u8 tcp_flags = 0;

        if ((flow->key.eth.type == htons(ETH_P_IP) ||
             flow->key.eth.type == htons(ETH_P_IPV6)) &&
            flow->key.ip.proto == IPPROTO_TCP &&
            likely(skb->len >= skb_transport_offset(skb) + sizeof(struct tcphdr))) {
                u8 *tcp = (u8 *)tcp_hdr(skb);
                tcp_flags = *(tcp + TCP_FLAGS_OFFSET) & TCP_FLAG_MASK;
        }

        spin_lock(&flow->lock);
        flow->used = jiffies;
        flow->packet_count++;
        flow->byte_count += skb->len;
        flow->tcp_flags |= tcp_flags;
        spin_unlock(&flow->lock);
}

```

- 如果該封包滿足(IP/IPV6,TCP)且TCP有額外的flag的話，就更新其TCP_FLAGS
- 更新該flow的相關資訊
- used(上次使用時間),單位是 **jiffies**
- counter.

### ovs_execute_actions
```c
/* Execute a list of actions against 'skb'. */
int ovs_execute_actions(struct datapath *dp, struct sk_buff *skb)
{
        struct sw_flow_actions *acts = rcu_dereference(OVS_CB(skb)->flow->sf_acts);
        struct loop_counter *loop;
        int error;

        /* Check whether we've looped too much. */
        loop = &__get_cpu_var(loop_counters);
        if (unlikely(++loop->count > MAX_LOOPS))
                loop->looping = true;
        if (unlikely(loop->looping)) {
                error = loop_suppress(dp, acts);
                kfree_skb(skb);
                goto out_loop;
        }

        OVS_CB(skb)->tun_key = NULL;
        error = do_execute_actions(dp, skb, acts->actions,
                                         acts->actions_len, false);

        /* Check whether sub-actions looped too much. */
        if (unlikely(loop->looping))
                error = loop_suppress(dp, acts);

out_loop:
        /* Decrement loop counter. */
        if (!--loop->count)
                loop->looping = false;

        return error;
}

```

- 先從flow_sf_acts中取出對應的actions(sw_flow_actions)
- 這邊會限制執行 **do_execute_actions**的次數，設計理念尚未明瞭。
- 呼叫 **do_execute_actions**來做後續的處理

### do_execute_actions
``` c
static int do_execute_actions(struct datapath *dp, struct sk_buff *skb,
                        const struct nlattr *attr, int len, bool keep_skb)
{
        /* Every output action needs a separate clone of 'skb', but the common
         * case is just a single output action, so that doing a clone and
         * then freeing the original skbuff is wasteful.  So the following code
         * is slightly obscure just to avoid that. */
        int prev_port = -1;
        const struct nlattr *a;
        int rem;

        for (a = attr, rem = len; rem > 0;
             a = nla_next(a, &rem)) {
                int err = 0;

                if (prev_port != -1) {
                        do_output(dp, skb_clone(skb, GFP_ATOMIC), prev_port);
                        prev_port = -1;
                }

                switch (nla_type(a)) {
                case OVS_ACTION_ATTR_OUTPUT:
                        prev_port = nla_get_u32(a);
                        break;

                case OVS_ACTION_ATTR_USERSPACE:
                        output_userspace(dp, skb, a);
                        break;

                case OVS_ACTION_ATTR_PUSH_VLAN:
                        err = push_vlan(skb, nla_data(a));
                        if (unlikely(err)) /* skb already freed. */
                                return err;
                        break;

                case OVS_ACTION_ATTR_POP_VLAN:
                        err = pop_vlan(skb);
                        break;

                case OVS_ACTION_ATTR_SET:
                        err = execute_set_action(skb, nla_data(a));
                        break;

                case OVS_ACTION_ATTR_SAMPLE:
                        err = sample(dp, skb, a);
                        break;
                }

                if (unlikely(err)) {
                        kfree_skb(skb);
                        return err;
                }
        }

        if (prev_port != -1) {
                if (keep_skb)
                        skb = skb_clone(skb, GFP_ATOMIC);

                do_output(dp, skb, prev_port);
        } else if (!keep_skb)
                consume_skb(skb);

        return 0;

```

- flow的action都是透過**nlattr**來儲存，這是netlink相關的資料結構，因為 **user space**也會透過netlink的方式要求kernel直接處理封包，所以action都用 **nlattr**來處理
- 因為可以 output到多個port去，每次都會需要複製 **skb**，所以這邊使用 **prev_port**來處理只有一次 output的情況(不用複製)

## Not found
如果沒有找到該flow，kernel就會透過netlink的方式，把這個封包送到 **ovs-vswitched**去處理。

``` c
               struct dp_upcall_info upcall;
               upcall.cmd = OVS_PACKET_CMD_MISS;
               upcall.key = &key;
               upcall.userdata = NULL;
               upcall.portid = p->upcall_portid;
               ovs_dp_upcall(dp, skb, &upcall);
```

``` c
struct dp_upcall_info {
        u8 cmd;
        const struct sw_flow_key *key;
        const struct nlattr *userdata;
        u32 portid;
};

```

- 每個 **dp_upcall_info**都是透過 **netlink**的方式把資料送到 **userspace**,這邊要記錄
- 資料設定完畢後， 呼叫 **ovs_dp_upcall**來處理

``` c
int ovs_dp_upcall(struct datapath *dp, struct sk_buff *skb,
                  const struct dp_upcall_info *upcall_info)
{
        struct dp_stats_percpu *stats;
        int dp_ifindex;
        int err;

        if (upcall_info->portid == 0) {
                err = -ENOTCONN;
                goto err;
        }

        dp_ifindex = get_dpifindex(dp);
        if (!dp_ifindex) {
                err = -ENODEV;
                goto err;
        }

        if (!skb_is_gso(skb))
                err = queue_userspace_packet(ovs_dp_get_net(dp), dp_ifindex, skb, upcall_info);
        else
                err = queue_gso_packets(ovs_dp_get_net(dp), dp_ifindex, skb, upcall_info);
        if (err)
                goto err;

        return 0;

err:
        stats = this_cpu_ptr(dp->stats_percpu);

        u64_stats_update_begin(&stats->sync);
        stats->n_lost++;
        u64_stats_update_end(&stats->sync);

        return err;
}
```

 - 檢查 porrid(port number)是否正常
 - 取得該 datapath的index
 - 根據有使用 **gso**的話，就會特別處理，因為封包的長度比較大，會透過多次的 **queue_userspace_packet**來處理支援 **gso**的封包。
 - 如果發生error，意味者這個封包就不會有人處理了，因此把lost的值增加


### queue_userspace_packet

``` c

 static int queue_userspace_packet(struct net *net, int dp_ifindex,
                                  struct sk_buff *skb,
                                  const struct dp_upcall_info *upcall_info)
{
        struct ovs_header *upcall;
        struct sk_buff *nskb = NULL;
        struct sk_buff *user_skb; /* to be queued to userspace */
        struct nlattr *nla;
        int err;
        if (vlan_tx_tag_present(skb)) {
                nskb = skb_clone(skb, GFP_ATOMIC);
                if (!nskb)
                        return -ENOMEM;

                nskb = __vlan_put_tag(nskb, nskb->vlan_proto, vlan_tx_tag_get(nskb));
                if (!nskb)
                        return -ENOMEM;

                vlan_set_tci(nskb, 0);

                skb = nskb;
        }

        if (nla_attr_size(skb->len) > USHRT_MAX) {
                err = -EFBIG;
                goto out;
        }

        user_skb = genlmsg_new(upcall_msg_size(skb, upcall_info->userdata), GFP_ATOMIC);
        if (!user_skb) {
                err = -ENOMEM;
                goto out;
        }

        upcall = genlmsg_put(user_skb, 0, 0, &dp_packet_genl_family,
                             0, upcall_info->cmd);
        upcall->dp_ifindex = dp_ifindex;

        nla = nla_nest_start(user_skb, OVS_PACKET_ATTR_KEY);
        ovs_flow_to_nlattrs(upcall_info->key, upcall_info->key, user_skb);
        nla_nest_end(user_skb, nla);

        if (upcall_info->userdata)
                __nla_put(user_skb, OVS_PACKET_ATTR_USERDATA,
                          nla_len(upcall_info->userdata),
                          nla_data(upcall_info->userdata));

        nla = __nla_reserve(user_skb, OVS_PACKET_ATTR_PACKET, skb->len);

        skb_copy_and_csum_dev(skb, nla_data(nla));
        err = genlmsg_unicast(net, user_skb, upcall_info->portid);

out:
        kfree_skb(nskb);
        return err;
}

```

- 這邊產生 **generic netlink** 然後把資料設定完畢後，就送出到 **userspace**
- **genlmsg_**系列尚未完全瞭解，待補充


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