---
layout: post
title: OpenVSwitch - Basic Install
date: '2013-11-30 17:11'
comments: true
tags:
  - SDN
  - Network
  - OpenvSwitch
keywords: 'SDN,OpenvSwitch,OVS,Kernel'
abbrlink: 7298
---
Environment
-----------

- System: Ubuntu 12.04 TLS
- OpenVSwtich : v.20 [openvswitch](http://openvswitch.org/download/ "openvswitch ")
- Controller: Floodlight controller

<!--more-->

#**Install**#

OpenVSwitch
-----------
按照文件中的INSTALL 即可安裝完成，

- ./configure  (如果要安裝kernel module的話，`./configure --with-linux=/lib/modules/`uname -r`/build` 來建置)
- make
- make modules_install
- /sbin/modprobe openvswitch
- 用 `lsmod | grep openvswitch` 檢查kernel module 是否載入
- mkdir -p /usr/local/etc/openvswitch
- ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema (創造ovs-db)
- ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock \
                     --remote=db:Open_vSwitch,Open_vSwitch,manager_options \
                     --private-key=db:Open_vSwitch,SSL,private_key \
                     --certificate=db:Open_vSwitch,SSL,certificate \
                     --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert \
                     --pidfile --detach

- ovs-vsctl --no-wait init
- ovs-vswitchd --pidfile --detach

Network Environment
-------------------

- 主機板網路孔*1 + 4 port 網卡
- eth0 藉由internet與controller連接 (out-bound)
- 用ovs 創造一個虛擬的interface br0,把eth1, eth2, eth3, eth4加入  (in-bound)
- eth1 & eth4 分別連上兩台host
- host1: 192.168.122.100
- host2: 192.168.122.101

![圖片1.png](http://user-image.logdown.io/user/415/blog/415/post/164871/pLwj6W3SR4ypbWvDvAra_%E5%9C%96%E7%89%871.png)

Operation
---------
- ovs-vsctl add-br br0
- ovs-vsctl add-port br0 eth1
- ovs-vsctl add-port br0 eth2
- ovs-vsctl add-port br0 eth3
- ovs-vsctl add-port br0 eth4
- ovs-vsctl set-controller br0  tcp:x.x.x.x:6633

Controller side
---------------
- Switch 00:00:90:e2:ba:49:58:84 connected.
- Watch x.x.x.x:8080/ui/index.html to see switch

Ovs side
--------
- ovs-vsctl show
```
    Bridge "br0"
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        Port "eth2"
            Interface "eth2"
        Port "eth4"
            Interface "eth4"
        Port "br0"
            Interface "br0"
                type: internal
        Port "eth3"
            Interface "eth3"
        Port "eth1"
            Interface "eth1"
```

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