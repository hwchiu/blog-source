---
layout: post
title: 'Openvswitch source code(1)'
date: 2013-12-02 09:17
comments: true
categories: 
tags:
	- SDN
	- Network
	- OpenvSwitch
	- SourceCode
keywords: SDN,OpenvSwitch,OVS,Kernel
---
Openvswitch support two modes for user to config, user mode and kernel mode.
we will discuss the kernel mode in this article.

Software version: openvswitch v.20

<!--more-->


##datapath.c##

This file is the main part of the kernel module and it will be compiled to the datapath.ko.
```
module_init(dp_init);
module_exit(dp_cleanup);

MODULE_DESCRIPTION("Open vSwitch switching datapath");
MODULE_LICENSE("GPL");
MODULE_VERSION(VERSION);
```

The kernel will call its init function `dp_init` after the kernel module has been loaded.

The following is the work flow of the `dp_init`.

- ovs_workqueues_init()  
- ovs_flow_init()
- ovs_vport_init
- register_pernet_device(&ovs_net_ops);
- register_netdevice_notifier(&ovs_dp_device_notifier);
- dp_register_genl();
- schedule_delayed_work(&rehash_flow_wq, REHASH_FLOW_INTERVAL);  

###ovs_workqueues_init###
**workqueue.c**

``` c
static struct task_struct *workq_thread;
spin_lock_init(&wq_lock);
INIT_LIST_HEAD(&workq);
init_waitqueue_head(&more_work);

workq_thread = kthread_create(worker_thread, NULL, "ovs_workq");
if (IS_ERR(workq_thread))
return PTR_ERR(workq_thread);

wake_up_process(workq_thread);
```

- Initail the worker queue.
-	Create a kernel thread and the handler is **worker_thread**
- start the kernel thread by calling **wake_up_process**
  
``` c
static int worker_thread(void *dummy)
{
        for (;;) {
                wait_event_interruptible(more_work,
                                (kthread_should_stop() || !list_empty(&workq)));

                if (kthread_should_stop())
                        break;

                run_workqueue();
        }

        return 0;
}
```

- wait_event_interruptible make the thread hibernation and add into the queue more_work.
- The thread will wake up until the condition "kthread_should_stop() || !list_empty(&workq))" is true.
- It will call the `run_workqueue` after it wake up.

``` c  
static void run_workqueue(void)
{
        spin_lock_irq(&wq_lock);
        while (!list_empty(&workq)) {
                struct work_struct *work = list_entry(workq.next,
                                struct work_struct, entry);

                work_func_t f = work->func;
                list_del_init(workq.next);
                current_work = work;
                spin_unlock_irq(&wq_lock);

                work_clear_pending(work);
                f(work);

                BUG_ON(in_interrupt());
                spin_lock_irq(&wq_lock);
                current_work = NULL;
        }
        spin_unlock_irq(&wq_lock);
}

```

- Get the work from the workq list and call the fucntion.

###ovs_flow_init###
**flow.c**

``` c
/* Initializes the flow module.
 * Returns zero if successful or a negative error code. */
int ovs_flow_init(void)
{
        BUILD_BUG_ON(__alignof__(struct sw_flow_key) % __alignof__(long));
        BUILD_BUG_ON(sizeof(struct sw_flow_key) % sizeof(long));

        flow_cache = kmem_cache_create("sw_flow", sizeof(struct sw_flow), 0,
                                        0, NULL);
        if (flow_cache == NULL)
                return -ENOMEM;

        return 0;
}

```

- use the `kmem_cache_create` to create a kernel cache with size `sw_flow`

``` c
struct sw_flow {
        struct rcu_head rcu;
        struct hlist_node hash_node[2];
        u32 hash;

        struct sw_flow_key key;
        struct sw_flow_key unmasked_key;
        struct sw_flow_mask *mask;
        struct sw_flow_actions __rcu *sf_acts;

        spinlock_t lock;        /* Lock for values below. */
        unsigned long used;     /* Last used time (in jiffies). */
        u64 packet_count;       /* Number of packets matched. */
        u64 byte_count;         /* Number of bytes matched. */
        u8 tcp_flags;           /* Union of seen TCP flags. */
};
```
- This struct store the info of each flow, including count, flow_key and flow_mask.

###ovs_vport_init###
**vport.c"**
``` c
/**
 *      ovs_vport_init - initialize vport subsystem
 *
 * Called at module load time to initialize the vport subsystem.
 */
int ovs_vport_init(void)
{
        dev_table = kzalloc(VPORT_HASH_BUCKETS * sizeof(struct hlist_head),
                            GFP_KERNEL);
        if (!dev_table)
                return -ENOMEM;

        return 0;
}
```
- Use `kzalloc` malloc the memory from kernel.
 
 
###register_pernet_device(&ovs_net_ops)###
``` c
	register_pernet_device(&ovs_net_ops);
```

- Register a network device `ovs_net_ops`

``` c
static struct pernet_operations ovs_net_ops = {
        .init = ovs_init_net,
        .exit = ovs_exit_net,
        .id   = &ovs_net_id,
        .size = sizeof(struct ovs_net),
};
```

- ovs_net_ops inherent from `pernet_operations`, it should implement some function (init, exit)

``` c
static int __net_init ovs_init_net(struct net *net)
{
        struct ovs_net *ovs_net = net_generic(net, ovs_net_id);

        INIT_LIST_HEAD(&ovs_net->dps);
        INIT_WORK(&ovs_net->dp_notify_work, ovs_dp_notify_wq);
        return 0;
}
```

- Use `net_generic` get the pointer to ovs_net.
- Use `INIT_WORK` to create a worker and set the function (dp_notify_work) as its work.


``` c
struct ovs_net {
        struct list_head dps;
        struct vport_net vport_net;
        struct work_struct dp_notify_work;
};
```

- need to study later.

``` c 
void ovs_dp_notify_wq(struct work_struct *work)
{
        struct ovs_net *ovs_net = container_of(work, struct ovs_net, dp_notify_work);
        struct datapath *dp;

        ovs_lock();
        list_for_each_entry(dp, &ovs_net->dps, list_node) {
                int i;

                for (i = 0; i < DP_VPORT_HASH_BUCKETS; i++) {
                        struct vport *vport;
                        struct hlist_node *n;

                        hlist_for_each_entry_safe(vport, n, &dp->ports[i], dp_hash_node) {
                                struct netdev_vport *netdev_vport;

                                if (vport->ops->type != OVS_VPORT_TYPE_NETDEV)
                                        continue;

                                netdev_vport = netdev_vport_priv(vport);
                                if (netdev_vport->dev->reg_state == NETREG_UNREGISTERED ||
                                    netdev_vport->dev->reg_state == NETREG_UNREGISTERING)
                                        dp_detach_port_notify(vport);
                        }
                }
        }
        ovs_unlock();
}

```

- search datapathes and list all its vport.
- Delete the vport if its status is `UNREGISTERED` of `UNREGISTERING`.


###register_netdevice_notifier###

``` c
 register_netdevice_notifier(&ovs_dp_device_notifier);
```

- Register the network notification chain, it will call `ovs_dp_device_notifier` when event occur.


```c
struct notifier_block ovs_dp_device_notifier = {
        .notifier_call = dp_device_event
};

```

- `ovs_dp_device_notifier` contains a function pointer which point to `dp_device_event`.
- This function will be call when the notification has occur.

``` c
static int dp_device_event(struct notifier_block *unused, unsigned long event,
                           void *ptr)
{
        struct ovs_net *ovs_net;
        struct net_device *dev = ptr;
        struct vport *vport = NULL;

        if (!ovs_is_internal_dev(dev))
                vport = ovs_netdev_get_vport(dev);

        if (!vport)
                return NOTIFY_DONE;

        if (event == NETDEV_UNREGISTER) {
                ovs_net = net_generic(dev_net(dev), ovs_net_id);
                queue_work(&ovs_net->dp_notify_work);
        }

        return NOTIFY_DONE;
}

```

- Need to study.

###dp_register_genl###
**datapath.c**

```  c
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

- Register four types of gereric netlink (datapath, vport, flow, packet).
- You can see the detail info in **dp_genl_families**
- `genl_register_family_with_ops`  : register a generic netlink family with ops.

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


``` c
struct genl_family_and_ops {
        struct genl_family *family;
        struct genl_ops *ops;
        int n_ops;
        struct genl_multicast_group *group;
};

```

- A genl_family_and_ops contains a pointer to its family and a pointer to its operations.

``` c
static struct genl_family dp_datapath_genl_family = {
        .id = GENL_ID_GENERATE,
        .hdrsize = sizeof(struct ovs_header),
        .name = OVS_DATAPATH_FAMILY,
        .version = OVS_DATAPATH_VERSION,
        .maxattr = OVS_DP_ATTR_MAX,
        .netnsok = true,
         SET_PARALLEL_OPS
};



static struct genl_ops dp_datapath_genl_ops[] = {
        { .cmd = OVS_DP_CMD_NEW,
          .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
          .policy = datapath_policy,
          .doit = ovs_dp_cmd_new
        },
        { .cmd = OVS_DP_CMD_DEL,
          .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
          .policy = datapath_policy,
          .doit = ovs_dp_cmd_del
        },
        { .cmd = OVS_DP_CMD_GET,
          .flags = 0,               /* OK for unprivileged users. */
          .policy = datapath_policy,
          .doit = ovs_dp_cmd_get,
          .dumpit = ovs_dp_cmd_dump
        },
        { .cmd = OVS_DP_CMD_SET,
          .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
          .policy = datapath_policy,
          .doit = ovs_dp_cmd_set,
        },
};
```

- Take **dp_datapath_genl_ops** for example. when the event is **OVS_DP_CMD_NEW** it will call it function handler **ovs_dp_cmd_new**.


###schedule_delayed_work###

- Need to study

###other###

- pr_info is printk(KERN_INFO,pr_fmt(fmt), ##__VA_ARGS__)  
