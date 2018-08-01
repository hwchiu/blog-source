---
title: Install DRBD v9.0 on Ubuntu 16.04
tags:
  - System
  - Ubuntu
  - DRBD
keywords: 'DRBD,Ubuntu,Linux,System'
abbrlink: 19938
date: 2017-05-18 13:48:48
---

### Introduction
本篇文章主要講述如何再 Ubuntu 16.04 with kernel 4.4.3 的環境下安裝 drbd 9.0 並進行簡單的設定與操作。


### Install
這邊為了方便日後的研究，這邊安裝的方式是抓取 source code 下來，然後進行編譯安裝，由於 drbd v8.4.5 後將  module 以及相關的 utils 是分開在不同的 **git repostory**，所以我們會有兩個 **project** 來編譯及安裝。
首先到[官網](http://git.drbd.org/)的 git 首頁可以看到滿滿的 projects，這邊我們會需要的兩個 project 分別是 [drbd-9.0](http://git.drbd.org/drbd-9.0.git) 以及 [drbd-utils](http://git.drbd.org/drbd-utils.git)。
接下來就說明這兩個 project 要如何編譯及安裝
<!--more-->
#### drbd-9.0
此 **project** 負責的是 kernel module部分，所以在編譯時會需要 kernel source 來編譯，如果你是正常安裝的 ubuntu 16.04，系統內應該都已經有 source 可以用了，這部分不太需要額外設定即可，若有特定的 kernel version 想要使用，則記得要先將該 kernel source 抓下來，然後編譯的時候指定特定的 kernel source 路徑即可。
這方面可以參考[官方的文件說明](https://drbd15-staging.linbit.com/en/doc/users-guide-83/s-build-from-source#s-build-prepare-kernel-tree)

流程基本上就是
- clone git project
- build
- install

基本上此編譯此 project 的過程非常順利，再執行`make`完畢後，會顯示一段文字
我們可以知道若想要使用 drbd 9.0 的 kernel 版本，則我們的 **drbd-utils** 至少要 8.9.11 版本。

            Module build was successful.
    =======================================================================
      With DRBD module version 8.4.5, we split out the management tools
      into their own repository at http://git.linbit.com/drbd-utils.git
      (tarball at http://links.linbit.com/drbd-download)

      That started out as "drbd-utils version 8.9.0",
      and provides compatible drbdadm, drbdsetup and drbdmeta tools
      for DRBD module versions 8.3, 8.4 and 9.

      Again: to manage DRBD 9 kernel modules and above,
      you want drbd-utils >= 8.9.11 from above url.
    =======================================================================

最後執行 `make install` 將相關的 kernel module 給安裝到系統的路徑，然後透過檢視可以發現實際上安裝的 modules  有 **drbd.ko** 以及 **drbd_transport_tcp.ko**。
分別是整個 drbd 核心的部分，以及網路功能的部分，若是商業化版本還可以多看到 **drbd_transport_rdma.ko** 供 RDMA 使用。

整個步驟如下。
``` bash
git clone http://git.drbd.org/drbd-9.0.git
cd drbd-9.0
make
make install
```
#### drbd-utils
此 **project** 提供 **drbd user space** 的所有工具，包含了 **drbdadm**, **drbdsetup**等常用工具。
基本上流程也是滿順利的
1. clone git project
2. autogen
3. configure
4. build
5. install

透過 `autogen.sh` 產生好對應的 **configure** 檔案時，會有下列文字說明

    suggested configure parameters:
    # prepare for rpmbuild, only generate spec files
    ./configure --enable-spec
    # or prepare for direct build
    ./configure --prefix=/usr --localstatedir=/var --sysconfdir=/etc

這邊就建議依照他的說法去設定 **configure**，不然之後執行 **drbdadm up resource** 的時候會發現有些東西找不到，如果不想要建議舊版的 tools 的話，可以加上這兩個參數
**--without-83support** 以及 **--without-84support**
此外，如果最後再建置的時候發現 **documentation/v9** 一直建置不過，然後又不需要文件的話，可以加上下列參數 **--without-manual**

這邊要注意的就是在 **make** 的時候會需要 **xsltproc** 這個套件，所以若有發現錯誤顯示 **xsltproc: command not found**，則記得透過 **apt-get install xsltproc** 安裝該套件即可。

整個步驟如下。
``` bash
git clone http://git.drbd.org/drbd-utils.git
./autogen.sh
./configure --prefix=/usr --localstatedir=/var --sysconfdir=/etc --without-83support --without-84support --without-manual
make
make install
```

### Configure
drbd 使用 **drbd.conf** 來設定相關資訊，預設的存放位置是 **/usr/local/etc/drbd.conf**，若之前在 configure 時有透過 **--sysconfdir=/etc**，則該 configure 的預設位置是 **/etc/drbd.conf**。
這個 config 需要每一台要跑 drbd 的機器上都要有一份，所以當設定完畢後，請自行 copy 到另外一台。本文中假設有兩台機器，其 hostname 分別是 **node-1** 以及 **node-2**。

大致步驟如下
- 設定 /etc/hosts
- 設定 config
- 將 config 複製到所有機器

首先由於 **drbd** 設定 host的時候，會使用 `hostname` 去尋找對應的 host 欄位，所以建議先修改 **/etc/hosts** 將所有用到的 hostname 與其 ip 對應關係都寫上去。
加入下列資訊魚 **/etc/hosts**
``` 
10.0.0.15 node-1
10.0.0.16 node-2
```

接下來我們要設定 **drbd.conf**，假設我們要使用系統上的 /dev/nvme0n1 當作我們的 disk，提供出來的 block device 是 **/dev/drbd0**，則範例如下

``` conf
global { usage-count no; }
common { protocol C; }

resource r0 {
        on node-1 {
                device /dev/drbd0;
                disk /dev/nvme0n1;
                address 10.0.0.15:7788;
                meta-disk internal;
        }
        on node-2 {
                device /dev/drbd0;
                disk /dev/nvme0n1;
                address 10.0.0.16:7788;
                meta-disk internal;
        }
}
```

接下來可以透過 **scp** 之類的指令將該設定檔複製到另外一台 **node-2**，或是有任何方法都可以，只要確保兩台有一樣的資料即可。

### Run
設定檔都準備完成後，接下來要依賴 **drbdadm** 幫忙進行相關的設定
首先我們使用 `drbdadm create-md` 將該 resource 給建立起來，大概訊息如下

    You want me to create a v09 style flexible-size internal meta data block.
    There appears to be a v09 flexible-size internal meta data block
    already in place on /dev/nvme0n1 at byte offset 400088453120

    Do you really want to overwrite the existing meta-data?
    [need to type 'yes' to confirm] yes

    initializing activity log
    initializing bitmap (11924 KB) to all zero
    Writing meta data...
    New drbd meta data block successfully created.

接下來透過 **drbdadm up r0** 將整個 resource 運行起來，包含將 device bloack attach，建立網路連線等。
待**node-1**以及**node-2**執行好上述指令後，我們要將 **node1** 當作 primary，所以這時候再 **node-1** 上面執行 `drbdadm primary r0` 如此一來就會將 **node-1**上面的資料從給 mirror 到 **node-2**上了。

接下來應該可以透過下列指令觀察到一些狀態
- drbdadm cstate r0
    - 觀察網路連線狀態
- drbdadm dstate r0
    - 觀察 disk 的狀態
- drbdadm status r0
    - 觀察整體狀態，包含其他的node是 **primary**/**secondary**等
- drbd-overview
    - 顯示當前cluster內的狀態
- drbdsetup status r0 --verbose --statistics
    - 顯示當前 sync 統計資訊，譬如還有多少資料未sync
``` bash
r0 node-id:1 role:Primary suspended:no
    write-ordering:flush
  volume:0 minor:0 disk:UpToDate
      size:390699424 read:390700584 written:12390400 al-writes:2750 bm-writes:0 upper-pending:0 lower-pending:0 al-suspended:no blocked:no
  node-2 node-id:0 connection:Connected role:Secondary congested:no
    volume:0 replication:Established peer-disk:UpToDate resync-suspended:no
        received:0 sent:403089824 out-of-sync:0 pending:0 unacked:0
```

更多的指令用法可參考官方文件上的[說明](https://drbd15-staging.linbit.com/en/doc/users-guide-90/s-check-status)

### Test
為了確認是否真的有流量在兩個 node 之間運行，可以使用 `dd` 這個指令於 **node-1**上面去寫入資料，然後透過**drbdsetup status r0 --verbose --statistics**確認有產生大量的資料進行 sync

``` bash
dd if=/dev/zero of=/dev/drbd0 bs=1M count=10000
```

### Trouble Shooting
- 執行 `drbdadm primary r0` 出現錯誤 **State change failed: (-2) Need access to UpToDate data**
    - 執行 `drbdadm primary r0 --force` 強迫蓋掉
