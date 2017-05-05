---
layout: post
title: 'OpenvSwitch - hmap'
date: 2013-12-27 04:18
comments: true
categories: 
tags:
	- SDN
	- Network
	- OpenvSwitch
---
**hmap** 是一種hash bucket的資料結構，在OVS中到處都有弄到，譬如 **kernel space**中的flow_key就是透過這種結構來存放的。

示意圖如下

![hmap.png](http://user-image.logdown.io/user/415/blog/415/post/169371/hZKD65KuSJyQat4j7Qd6_hmap.png)

<!--more-->


每一個 **hmap_node**都存放一個hash值，相同hash值的人會透過單向link串起來
**hmap**擁有多個指標，指向每個hash的開頭，也就是所謂的bucket，所有的操作都要透過此結構


**hmap_node** 
``` c
struct hmap_node {
    size_t hash;                /* Hash value. */
    struct hmap_node *next;     /* Next in linked list. */
};
```

- 這個結構很簡單，紀錄了本身的 hash值，並且有一個指標指向下一個相同hash的node

**hmap**
``` c
struct hmap {
    struct hmap_node **buckets; /* Must point to 'one' iff 'mask' == 0. */
    struct hmap_node *one;
    size_t mask;
    size_t n;
};
```

- buckets 是一個 **pointer to pointer**, 指向各個不同hash value的開頭node.
- one 用途不明
- mask 搭配hash值可以得到對應的bucket
- 目前hmap中已經有多少個 **hmap_node**, n< 2*mask + 1.

**hmap.h/hmap.c**
關於hmap的操作大部分都定義在這兩個檔案內，有function也有marco.這邊節錄幾個來看


##insert##
``` c
static inline void
hmap_insert_fast(struct hmap *hmap, struct hmap_node *node, size_t hash)
{
    struct hmap_node **bucket = &hmap->buckets[hash & hmap->mask];
    node->hash = hash;
    node->next = *bucket;
    *bucket = node;
    hmap->n++;
}
```

- 先利用此hash與mask找到對應的 bucket, 值得注意的是這邊拿到的也是一個 **pointer to pointer**
- node的 next 指向 bucket所指向的第一個node，然後bucket則改成指向node，結論就是會把這個node從前面插入


``` c
#define HMAP_FOR_EACH_WITH_HASH(NODE, MEMBER, HASH, HMAP)               \
    for (ASSIGN_CONTAINER(NODE, hmap_first_with_hash(HMAP, HASH), MEMBER); \
         NODE != OBJECT_CONTAINING(NULL, NODE, MEMBER);                  \
         ASSIGN_CONTAINER(NODE, hmap_next_with_hash(&(NODE)->MEMBER),   \
                          MEMBER))
```

- 這是一個用來搜尋的 marco,使用到了 **ASSIGN_CONTAINER** 以及 **OBJECT_CONTAINING**兩個marco
- 呼叫 **ASSIGN_CONTAINER** 取得在hmap中含有特定 hash的第一個 **hmap_node**
- **OBJECT_CONTAINING** 回傳一個NULL物件
- 每次都透過 **hmap_next_with_hash** 取得相同hash下的下一個node

``` c
static inline struct hmap_node *
hmap_next_with_hash__(const struct hmap_node *node, size_t hash)
{
    while (node != NULL && node->hash != hash) {
        node = node->next;
    }
    return CONST_CAST(struct hmap_node *, node);
}

static inline struct hmap_node *
hmap_next_with_hash(const struct hmap_node *node)
{
    return hmap_next_with_hash__(node->next, node->hash);
}
```


