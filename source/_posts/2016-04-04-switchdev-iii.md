---
layout: post
title: 'Switchdev IIII'
date: 2016-04-04 08:38
comments: true
categories: 
tags:
	- System
	- Kernel
	- Switchdev
	- Network

---
Introduction
------------
本篇文章主要會專注於 switchdev 本身的實作上，包含了其結構以及提供的 API 等。
  

<!--more-->


  
Structure
---------
###Transaction###
```
struct switchdev_trans_item {
        struct list_head list;
        void *data;
        void (*destructor)(const void *data);
};

struct switchdev_trans {
        struct list_head item_list;
        bool ph_prepare;
};

static inline bool switchdev_trans_ph_prepare(struct switchdev_trans *trans)
{
        return trans && trans->ph_prepare;
}

static inline bool switchdev_trans_ph_commit(struct switchdev_trans *trans)
{
        return trans && !trans->ph_prepare;
}
```
- Switchdev [實作](https://patchwork.ozlabs.org/patch/519719/)了一種 trans 的機制，對於 hardware switch 進行一些寫入的動作，如 set/add 時，該動作會被拆成兩部份，分別是 prepare/commit 兩部分。
- 一開始會先將 **ph_prepare** 給設定為 `true`，然後寫入的資料傳給 driver，讓 driver 知道這次的寫入只是用來確認可行性而已，如果 driver 確定可以寫入後，會將 **ph_prepare** 變為 `false` 後，再次要求 driver 將真正的資料給寫入。


###Attribute###
```
enum switchdev_attr_id {
        SWITCHDEV_ATTR_ID_UNDEFINED,
        SWITCHDEV_ATTR_ID_PORT_PARENT_ID,
        SWITCHDEV_ATTR_ID_PORT_STP_STATE,
        SWITCHDEV_ATTR_ID_PORT_BRIDGE_FLAGS,
        SWITCHDEV_ATTR_ID_BRIDGE_AGEING_TIME,
        SWITCHDEV_ATTR_ID_BRIDGE_VLAN_FILTERING,
};
```

  此 enum 用來定義 switch attribute 的種類，當 switch driver 收到一些如 get 的指令時，會根據該 attribute的種類回傳特定資料

```
struct switchdev_attr {
        struct net_device *orig_dev;
        enum switchdev_attr_id id;
        u32 flags;
        union {
                struct netdev_phys_item_id ppid;        /* PORT_PARENT_ID */
                u8 stp_state;                           /* PORT_STP_STATE */
                unsigned long brport_flags;             /* PORT_BRIDGE_FLAGS */
                u32 ageing_time;                        /* BRIDGE_AGEING_TIME */
                bool vlan_filtering;                    /* BRIDGE_VLAN_FILTERING */
        } u;
};
```
實際上用來紀錄 switch attribute 的結構
- **net_device** 來記錄是哪個目標 device
- id 如前述所說的種類
- flags 目前有三種值
```
#define SWITCHDEV_F_NO_RECURSE          BIT(0)
#define SWITCHDEV_F_SKIP_EOPNOTSUPP     BIT(1)
#define SWITCHDEV_F_DEFER               BIT(2)
```
- u 則是用來存放該 id 所代表的值

###Object###
```
enum switchdev_obj_id {
        SWITCHDEV_OBJ_ID_UNDEFINED,
        SWITCHDEV_OBJ_ID_PORT_VLAN,
        SWITCHDEV_OBJ_ID_IPV4_FIB,
        SWITCHDEV_OBJ_ID_PORT_FDB,
        SWITCHDEV_OBJ_ID_PORT_MDB,
};
```
  此 enum 用來記錄該 switch object 的種類，
```
struct switchdev_obj {
        struct net_device *orig_dev;
        enum switchdev_obj_id id;
        u32 flags;
};
```
  此結構記錄 type， net_device 以及 flag，對應種類的數值因為太過於廣泛，所以此 structure 會再被其他的結構給包起來。
```
struct switchdev_obj_port_vlan {
        struct switchdev_obj obj;
        u16 flags;
        u16 vid_begin;
        u16 vid_end;
};
struct switchdev_obj_ipv4_fib {
        struct switchdev_obj obj;
        u32 dst;
        int dst_len;
        struct fib_info fi;
        u8 tos;
        u8 type;
        u32 nlflags;
        u32 tb_id;
};
struct switchdev_obj_port_fdb {
        struct switchdev_obj obj;
        unsigned char addr[ETH_ALEN];
        u16 vid;
        u16 ndm_state;
};
struct switchdev_obj_port_mdb {
        struct switchdev_obj obj;
        unsigned char addr[ETH_ALEN];
        u16 vid;
};
```
由上面可以觀察到，目前已經實作了四種的 switchdev obj，分別是 vlan 的設定， L2 的 FDB/MDB 以及 L3 的 FIB.
###Operation###
```
/**
 * struct switchdev_ops - switchdev operations
 *
 * @switchdev_port_attr_get: Get a port attribute (see switchdev_attr).
 *
 * @switchdev_port_attr_set: Set a port attribute (see switchdev_attr).
 *
 * @switchdev_port_obj_add: Add an object to port (see switchdev_obj_*).
 *
 * @switchdev_port_obj_del: Delete an object from port (see switchdev_obj_*).
 *
 * @switchdev_port_obj_dump: Dump port objects (see switchdev_obj_*).
 */
struct switchdev_ops {
        int     (*switchdev_port_attr_get)(struct net_device *dev,
                                           struct switchdev_attr *attr);
        int     (*switchdev_port_attr_set)(struct net_device *dev,
                                           const struct switchdev_attr *attr,
                                           struct switchdev_trans *trans);
        int     (*switchdev_port_obj_add)(struct net_device *dev,
                                          const struct switchdev_obj *obj,
                                          struct switchdev_trans *trans);
        int     (*switchdev_port_obj_del)(struct net_device *dev,
                                          const struct switchdev_obj *obj);
        int     (*switchdev_port_obj_dump)(struct net_device *dev,
                                           struct switchdev_obj *obj,
                                           switchdev_obj_dump_cb_t *cb);
};
```

此結構被加入到 **struct net_device**內，所以 hardware switch driver 在創建 net_divce 時，要順便對該結構進行初始化，這樣對應的 function pointer 才有辦法在適當的時機被執行，這部分可以參考 rocker driver。
```
dev->switchdev_ops = &rocker_port_switchdev_ops;
```

###Notifier###
```
enum switchdev_notifier_type {
        SWITCHDEV_FDB_ADD = 1,
        SWITCHDEV_FDB_DEL,
};

struct switchdev_notifier_info {
        struct net_device *dev;
};

struct switchdev_notifier_fdb_info {
        struct switchdev_notifier_info info; /* must be first */
        const unsigned char *addr;
        u16 vid;
};
```
Notifier 是用來讓 hardware switch 通知 linux kernel 用的，目前只有實作 FDB 的部分。
當 hardware switch 的 FDB offload 有變化時(ADD/DEL)，要透過這個方式一路通知道 linux kernel 去，這樣的話使用 `brctl show` 指令去看的時候，就可以看到即時的狀態變化。


Implementation
--------------
###SwitchDev Port Attribute###
```
int switchdev_port_attr_get(struct net_device *dev,
                            struct switchdev_attr *attr);
int switchdev_port_attr_set(struct net_device *dev,
                            const struct switchdev_attr *attr);
```
這兩個 function 是用來處理 attribute 的，其處理邏輯類似，基本上都按照下列走法

```
const struct switchdev_ops *ops = dev->switchdev_ops;

if (ops && ops->switchdev_port_attr_get)
	return ops->switchdev_port_attr_get(dev, attr);

if (attr->flags & SWITCHDEV_F_NO_RECURSE)
	return err;
  
netdev_for_each_lower_dev(dev, lower_dev, iter) {
	/* do something */
}
```
1. 先判斷該 device 是否有實作 switchdev_ops,若有的話則直接呼叫 fptr 來處理.
	- 參考 rocker driver.
```
static const struct switchdev_ops rocker_port_switchdev_ops = {
	.switchdev_port_attr_get        = rocker_port_attr_get,
	.switchdev_port_attr_set        = rocker_port_attr_set,
}
...
dev->switchdev_ops = &rocker_port_switchdev_ops;
...
```
2. 判斷該 device 是否有被設定不需要遞迴往下尋找，若有的話則直接結束
3. 因為 switch port 可能是屬於 bond/team/vlan 等 device 底下，所以若直接操作上層的 device 是沒有辦法碰到 switch port 的，這邊會使用 [netdev_for_each_lower_dev](https://lists.ubuntu.com/archives/kernel-team/2014-June/043300.html) 來嘗試抓取到底下所有的 device。
	- 對於 get/set 來說，會針對底下每個 device 嘗試去 get/set 其 attribute.

###SwitchDev Port Object operation###
```
int switchdev_port_obj_add(struct net_device *dev,
                           const struct switchdev_obj *obj);                    
int switchdev_port_obj_del(struct net_device *dev,
                           const struct switchdev_obj *obj);
int switchdev_port_obj_dump(struct net_device *dev, struct switchdev_obj *obj,
                            switchdev_obj_dump_cb_t *cb);
```
這三個 function 都是用來處理 object 的，其運作邏輯也類似
```
const struct switchdev_ops *ops = dev->switchdev_ops;

if (ops && ops->switchdev_port_obj_add)
                return ops->switchdev_port_obj_add(dev, obj, trans);
netdev_for_each_lower_dev(dev, lower_dev, iter) {
	/* do something */
}
```
1. 先檢查該 device 是否有實作 switchdev_ops,若有就呼叫對應的 function 來處理
2. 遞迴存取底下所有的 device (bond/team/vlan), 針對每個 device 都跑一次對應的結果。
3. obj_dump 的部分還會傳入一個 **call back** function, 目前看到的只有兩個實作，分別是 **switchdev_port_obj_dump** 以及 **switchdev_port_vlan_dump_cb**。 兩者都要搭配另外一個 `switchdev_port_xxx_dump** 的結構來使用，目前感覺用途不是很 general.
	- fdb 的 dump 與 **rfnetlink** 有關係，要搭配 **ndo_fdb_dump** 使用。user space tool 透過 netlink 來問 fdb 的資料時，會透過此 cb 將對應的內容填入到 netlink header 中，最後再一路送回 user space 去檢查。
  - vlan 的部分則是 **rfnetlink** 在使用的，會先呼叫到 **ndo_bridge_getlink**, 最後跑到 **ndo_dflt_bridge_getlink** 才會使用，ndo (network device operation) 的 netlink 操作有必要再多花一些時間去瞭解了。

### Port Bridge###
```
int switchdev_port_bridge_getlink(struct sk_buff *skb, u32 pid, u32 seq,
                                  struct net_device *dev, u32 filter_mask,
                                  int nlflags);
int switchdev_port_bridge_setlink(struct net_device *dev,
                                  struct nlmsghdr *nlh, u16 flags);
int switchdev_port_bridge_dellink(struct net_device *dev,
                                  struct nlmsghdr *nlh, u16 flags);
```
這三個 function 是用來操作 bridge port attribute 的，基本上都是被設定成 **ndo_bridge_xxx** 的 handler。
目前可以參考的範例應該是使用 **br** 這個與 **ip** 類似的 user-space tool.
詳細說明可以參考此 [link](http://comments.gmane.org/gmane.linux.network/232104)


###FDB Operations###
```
int switchdev_port_fdb_add(struct ndmsg *ndm, struct nlattr *tb[],
                           struct net_device *dev, const unsigned char *addr,
                           u16 vid, u16 nlm_flags);
int switchdev_port_fdb_del(struct ndmsg *ndm, struct nlattr *tb[],
                           struct net_device *dev, const unsigned char *addr,
                           u16 vid);
int switchdev_port_fdb_dump(struct sk_buff *skb, struct netlink_callback *cb,
                            struct net_device *dev,
                            struct net_device *filter_dev, int idx);
```                           
這三個 function 是用來操作 fdb 的，當上層走 **rtnetlink** 中的 **ndo_fdb_xxx** type 時，就會觸發對應的 function handler，這些 function 最後都會呼叫到對應的 **switchdev_port_obj_xxx**。

關於整個 FDB 的操作，可以用下列這張圖來總結
![](https://lh3.googleusercontent.com/-sk760sgtc28/VwO9Wle0NKI/AAAAAAAAFOo/QrHADU1hDps5c5SmGBwU-nd54ZSJ7UjpQCCo/s720-Ic42/FDB.png)
- 藍線代表的是 Notifer，當 Rocket 在 FDB 有任何變更時，會一路通知到 Linux Kernel 去，以確保 FDB 資料一致。  
- 圖中紅線代表的是走 ndo 系列的 netlink event，會直接跟 Rocker 溝通，因此透過 `br` 此指令去修改 FDB entry時，會先走紅線到 Rocker 去，接者走藍線去通知 Kernel 同步 FDB。
- 當透過`brctl`指令去操作時，這邊目前能做的只有部分 attribute/obj 的修改，如 STP 的狀態，此時則會一路從 switchdev 的核心傳到 Rocker 去處理。
- 基本上 MDB 的操作則簡單很多，與 FIB 比較類似。


###FIB Operations###
```
int switchdev_fib_ipv4_add(u32 dst, int dst_len, struct fib_info *fi,
                           u8 tos, u8 type, u32 nlflags, u32 tb_id);
int switchdev_fib_ipv4_del(u32 dst, int dst_len, struct fib_info *fi,
                           u8 tos, u8 type, u32 tb_id);
void switchdev_fib_ipv4_abort(struct fib_info *fi);
```
- 這三個 function 是用來操作 IPv4 FIB offload 的，不同於 FDB，此 offload rule 本身的學習只能靠 linux kernl 來管理，當 kernel 決定要針對特定 FIB route 處理時，會呼叫上述的 add/del 將相關的 FIB router 給加入到 hardware switch 中。

```
err = switchdev_fib_ipv4_add(key, plen, fi,
      new_fa->fa_tos,
      cfg->fc_type,
      cfg->fc_nlflags,
      tb->tb_id);
if (err) {
      switchdev_fib_ipv4_abort(fi);
      kmem_cache_free(fn_alias_kmem, new_fa);
      goto out;
}
```
- 當執行失敗的時候，會呼叫 abort 將 rules 給全部清空，並且將 IPv4 offload 給關閉
	- 這部分還有待加強，由註解也可以看出來
  
``` 
 /* There was a problem installing this route to the offload
 * device.  For now, until we come up with more refined
 * policy handling, abruptly end IPv4 fib offloading for
 * for entire net by flushing offload device(s) of all
 * IPv4 routes, and mark IPv4 fib offloading broken from
 * this point forward.
 */  
```

- 而目前在加入 rules 的部分，也有針對條件去篩選，並非所有的 FIB 都會被加入
```
#ifdef CONFIG_IP_MULTIPLE_TABLES
        if (fi->fib_net->ipv4.fib_has_custom_rules)
                return 0;
#endif

        if (fi->fib_net->ipv4.fib_offload_disabled)
                return 0;
```
關於整個 FIB 的操作，可以用下列這張圖來總結
![](https://lh3.googleusercontent.com/-6xvPI9pzSrQ/VwO-v0MUQ5I/AAAAAAAAFPA/4P4aYSo6pfMzzPLacKWG-lfICRtn087xQCCo/s720-Ic42/FIB.png)
- 相對於 FDB，非常的簡單，只有 kernel 主動去加入 Rocker 而已
- 目前 ndo_xxx_ooo 系列的操作中，還沒有看到 FIB 相關的，大部分都是 bridge/vlan/macvlan 等。

###Notifier###
```
int register_switchdev_notifier(struct notifier_block *nb);
int unregister_switchdev_notifier(struct notifier_block *nb);
int call_switchdev_notifiers(unsigned long val, struct net_device *dev,
                             struct switchdev_notifier_info *info);
```

- 基本上就是用讓 hardware switch driver 呼叫的，當 switchdevb 有任何更動需要讓上層知道時就會呼叫 **call_switchdev_notifiers**，此時所有透過 **register_switchdev_notifier** 註冊的 handler 都會去處理
- 目前有透過的 **register_switchdev_notifier** 註冊的只有 bridge(br.c), 目的是用來同步 FIB 資訊。
