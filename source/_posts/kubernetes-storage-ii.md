---
title: NFS 於 Kubernetes 內的各種應用
keywords: 'Kubernetes,NFS,StorageClass'
abbrlink: 58353
date: 2018-12-31 09:44:51
tags:
  - Storage
  - Kubernetes
description: Network FileSystem(NFS) 是一個普遍常用且方便的檔案系統, Kubernetes 本身也有支援 NFS 的使用，然而 NFS 本身的諸多限制以及常見的用法要如何跟 Kubernetes 完美的整合，各種 ExportPath 以及對應的 UNIX 檔案權限，甚至是 Kubernetes 所提供的容量限制，存取模式限制等。本文會針對 NFS 常見的使用方式介紹使用，並且介紹上述等常見功能如果要與 Kubernetes 整合會遇到的各種問題。此外本文也會介紹如何透過第三方的專案來提供 StorageClass 此動態分配的方式與 NFS 結合，提供另外一種使用情境。

---

# Preface

在前一篇文章中 [Kubernetes X Storage (I)
](https://www.hwchiu.com/kubernetes-storage-i.html) 已經跟大家介紹過基本的儲存概念，包含了 `PV/PVC/StorageClass` 等基本元件。

而本篇文章則會跟大家分享一下在 `Kubernetes` 內可以如何使用`NFS (Network File System)`，主要會涵蓋兩大主題，其中一個是基於 `Kubernetes` 原生支援的 `NFS` 使用方式，透過 `PV/PVC` 或是 `Pod` 內直接透過 `VolumeMounts` 來描述 `NFS` 的資訊來使用。
另外一種則是會透過 `kubernetes-incubator` 內相關的專案，譬如 `NFS-Provisioner`  或是 `NFS-Client-Provisaioner` 等來介紹關於 `StorageClass` 以及 `NFS` 互相整合的使用方式。

網路上關於 `NFS` 的用法非常多種，大致上分成
1. 透過 Volume 的架構使用 `NFS` 作為背後的儲存容器
2. 透過 Pod/Deployment 的方式於 `Kubernetes` 內架設一個 `NFS Server`
    - 通常這種範例，背後還會接譬如 Public Cloud Disk當作 NFS Server 的後端儲存來源。
4. 上述兩種混合，一邊當 NFS Client(Volume), 一邊當 Server (NFS Server)

而本文介紹的方式基本上會概括(1)/(2) 兩種用法。


# NFS Server
本文不會介紹太多關於 `NFS Server` 的建置，這部分請讀者自行完成，當建置完成後，可以透過 `showmount` 的方式來確認該 `NFS Server` 有正確的 `export` 出相關的資料夾以及權限。


本文的範例 `NFS Server` 資訊如下
IP: `172.17.8.101`
ExportPath: `/nfsshare`

```=bash
vagrant@vortex-dev:~$ showmount -e 172.17.8.101
Export list for 172.17.8.101:
/nfsshare
```

# Nativer Support
對於 `NFS` 來說， 官方有提供了一些使用範例來介紹如何使用，詳細的可以參考下列連結 [kubernetes/examples/nfs](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs).

## Pod
如果是要在 `Pod` 內直接使用 `NFS`， 非常簡單，只要在 `Volume` 的欄位描述相關的資訊即可。
這種使用方式就是簡單，但是不方便，每次都要描述相關的 `IP/Path` 等資訊，所以一般使用上還是會採用 `PV/PVC` 的架構。
```=yaml
spec:
  containers:
  - name: busybox
    image: hwchiu/netutils:latest
    volumeMounts:
      - name: nfs-volume
        mountPath: /nfs
  volumes:
    - name: nfs-volume
      nfs:
        server: 172.17.8.101
        path: /nfsshare
```

## PV/PVC
如果要使用 `PV/PVC` 的架構，流程如下
1. 創立 `PV` 資源，與事先架設好的 `NFS Server` 綁定
2. 創立 `PVC`，去尋找符合需求的 `PV`
3. 創立 `Pod`, 透過 `VolumeMounts` 的方式去掛載創立好的 `PVC`.

### PV/PVC/Pod

`PV` 必須要描述跟 `NFS Server` 最直接的資訊，譬如 IP 位置以及相關的路徑
```=yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 1Mi
  accessModes:
    - ReadOnlyMany
  nfs:
    server: 172.17.8.101
    path: "/nfsshare"
```

`PVC` 中直接因為這個範例很簡單，也沒有其他的 `PV` 存在，因此只要條件滿足就可以選擇到對應的 `PV` 並且連接起來

```=yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadOnlyMany
  storageClassName: ""
  resources:
    requests:
      storage: 1Mi
```

`Pod` 中透過`Volumes` 的資訊去綁定對應的 `PVC` 即可

```=yaml
apiVersion: v1
kind: Pod
metadata:
  name: hwchiu
  labels:
    app: hwchiu
spec:
  containers:
  - name: busybox
    image: hwchiu/netutils:latest
    volumeMounts:
      - name: nfs-volume
        mountPath: /nfs
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: nfs
```

基本上按照上述範例，就可以順利在 `Pod` 裡面的 `/nfs` 資料夾使用到外部 `NFS` 儲存伺服器的資料了

## Limitation
### Description
`NFS` 使用起來看起來非常順利簡單，實際上卻有一些限制，接下來好好探討一下這些限制

在上述的 `PV` 的創建過程中，我們可以看到我設立了
```=yaml
  capacity:
    storage: 1Mi
  accessModes:
    - ReadOnlyMany
```

然而實際上，我們到真正的 `Pod` 裡面執行下列指令，卻是可以正常執行的
```=bash
dd if=/dev/zero of=/nfs/ii count=40960
```
該指令會在 `/nfs/` 底下寫入一個將近 `200Mb` 的檔案。
這個操作完全忽略了上述的設定
1. 大小 `1Mi`
2. `ReadOnly` 的權限設定

這部分主要是因為 `NFS` 本身架構導致的結果。
首先， `NFS` 本身並不是根據`需求`而`動態分配空間`出來的一種檔案系統，本身檔案系統有多大，你可以存取的人就有多大可以使用。
因此過往在使用經驗上，通常都會仔細設計每個分享出來的資料夾架構以及透過其他的系統工具來限制該資料夾的能夠使用的空間大小。

此外，對於 `NFS` 來說，控管權限的部分分成兩個部分來看
1. MountOption
2. Unix 檔案權限系統 (0777)

首先，在 `MountOption` 的部分，我們可以在 `NFS Server/NFS Client` 的部分設定 `RO/RW` 等不同的權限，但是這些權限範圍過大，基本上就是控制該 `NFS Client` 是否有讀寫的權限而已

至於 Unix 檔案權限系統方面，基本上就是根據使用者的 `UID/GID` 以及該檔案本身的 `UID/GID/OTHER`相關的的權限來控制。

所以有一個很有趣的現象就是，假設你什麼都沒有處理的情況下，你叫起的 `Pod` 預設都是以 `root` 的身份去執行，意味該使用者擁有對 `NFS` 分享資料夾下所有檔案的所有權利，`Read/Write/Rename/Delete...` 都可以執行。

所以要好好地針對 `NFS` 的檔案權限去使用的人，必須要有
1. 完整了解整個 UNIX 檔案權限的知識
2. 完整規劃不同權限 `UID/GID` 等架構
3. 每個對應的 `Pod` 運行的時候都要處理對應的 `RunAsUser` 等設定，來切換對應的使用者

當你完成了上面這些設定，你就會發現實際上還是不能通，最主要的問題是在`UNIX` 裡面， `UID/GID` 實際上是一堆數字，該檔案權限系統比對的也是數字。

但是在你的 `Pod` 裡面，你的 `/etc/password` 不一定有該數字對應的使用者資料，所以你今天透過 `RunAsUser` 去換一個使用者名稱 `hwchiu` 來運行這個 `Pod`.

但是因為該 `Pod` 內並沒有找不到這個使用者的名稱，也沒辦法找到對應的 `UID`. 最後你就會被系統給強制轉換成 `Root`. 所以一切的檔案權限又回到原點了。

這部分非常麻煩處理，過往可能會想要透過 `NIS(YP)` 或是修改 `/etc/nsswitch.con` 的方式來調整查詢的順序。 但是這些舉動對於 `Pod` 來說非常麻煩及極度困難。

因此有想要完整的在 `Kubernetes` 提供 `NFS Clinet` 服務且也要支援外部權限系統的人，要好好的思考這些步驟該怎麼處理。


### Kubernetes
針對上述的限制，`kubernetes` 內也有一些額外的選項可以處理。
首先關於 `ReadOnly` 這些的選項，我們可以透過下列的參數
```=bash
   nfs:
    server: 172.17.8.101
    path: "/nfsshare"
    readOnly: true
```
讓 `Kubernetes` 知道`NFS`在掛載的時候可以請幫忙提供 `RO` 的參數。

接下來在使用 `PVC` 的時候也要給予類似的參數
```=bash
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: nfs
        readOnly: true
```

這樣的話我們在使用的 `Pod` 內使用`mount`指令可以觀察到 `RO`而非`RW`的選項
```=bash
root@hwchiu:/# mount | grep nfs
172.17.8.101:/nfsshare on /nfs type nfs4 (ro,relatime,vers=4.0,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=172.17.8.101,local_lock=none,addr=172.17.8.101)
root@hwchiu:/# touch /nfs/test
touch: cannot touch '/nfs/test': Read-only file system
root@hwchiu:/#
```

# Kubernetes-incubator: nfs-client
前面我們提到了透過 `PV/PVC` 的方式來事先設定好相關的儲存資源，並且分配讓對應的 `Pod` 來使用。

這種情況下，基本上每個 `Pod` 都可以共享相同的資料夾，並且看到相同的內容。如果該 `NFS Server` 有不一樣的 `Export Path` 且有不同的用途，通常就要創立不同的 `PV/PVC` 組合來供最上層的 `Pod` 來使用。

如果今天想要透過 `StorageClass` 這種動態分配的方式來使用 `NFS Server` 有沒有辦法?
預設的 `Kubernetes` 是沒有提供這類型的功能，但是我們可以透過額外的套件功能來幫我們滿足這個功能

也就是接下來要介紹的 `kubernetes-incubator` 內的專案 `NFS-Provisioner: nfs-client`

詳細的資訊可以參考其 [Kubernetes-Incubator External-Storage NFS-Client](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)

## Introduction
該專案的目標很簡單，讓你可以透過 `StorageClass` 動態產生 `PV/PVC` 供上層的 `Pod` 使用.

其原理很簡單，其實就是透過 `Kubernetes` 內建的 `Provisioner` 介面去額外實現一個 `Provisioner Controller`.

接下來當我們部署一個 `StorageClass` 的時候，可以於 `Provisioner` 的欄位去指向事先創立好的 `Provisioner`. 這樣當未來有任何 `PVC` 的需求要指向 `StorageClass`, 最後就會將該請求傳送到該 `Deploykent(Provisioner)` 內去處理。


### Installation
我們最主要需求就是運行起一個 `Provisioner Controller`. 這部分 `nfs-client` 有提供 `helm` 來安裝
```=bash
helm install stable/nfs-client-provisioner \
     --name nfs-client
     --set nfs.server=172.17.8.101 \
     --set nfs.path=/nfsshare
```

這邊我們需要針對 `nfs.server` 以及 `nfs.path` 去進行設定。當一切安裝完畢後，我們透過 `helm status nfs-client` 來觀察一下所有安裝的 `kubernetes` 資源
```=bash
~$ helm status nfs-client
LAST DEPLOYED: Sun Dec 30 15:16:04 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ServiceAccount
NAME                               AGE
nfs-client-nfs-client-provisioner  1m

==> v1/ClusterRole
nfs-client-nfs-client-provisioner-runner  1m

==> v1/ClusterRoleBinding
run-nfs-client-nfs-client-provisioner  1m


==> v1/Role
leader-locking-nfs-client-nfs-client-provisioner  1m

==> v1/RoleBinding
leader-locking-nfs-client-nfs-client-provisioner  1m

==> v1/Deployment
nfs-client-nfs-client-provisioner  1m

==> v1/Pod(related)

NAME                                               READY  STATUS   RESTARTS  AGE
nfs-client-nfs-client-provisioner-986bcfb76-svqb4  1/1    Running  0         1m

==> v1/StorageClass

NAME        AGE
nfs-client  1m

```

這邊我們可以觀察的是 `StorageClass` 以及 `Deployment`.
```=bash
~$ kubectl describe  storageclass nfs-client
Name:                  nfs-client
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           cluster.local/nfs-client-nfs-client-provisioner
Parameters:            archiveOnDelete=true
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

可以看到 `Provisioner` 這邊指定的是 `cluster.local/nfs-client-nfs-client-provisioner`, 而這個數值我們可以透過檢視該 `Deployment` 的環境變數來觀察到
```=bash
~$ kubectl describe deployment nfs-client-nfs-client-provisioner | grep PRO
      PROVISIONER_NAME:  cluster.local/nfs-client-nfs-client-provisioner
```

接下來我們就可以透過部署 `PVC` 以及對應的 `Pod` 來使用了
```=bash
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-client"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

```=yaml
apiVersion: v1
kind: Pod
metadata:
  name: hwchiu
  labels:
    app: hwchiu
spec:
  containers:
  - name: busybox
    image: hwchiu/netutils:latest
    volumeMounts:
      - name: nfs-volume
        mountPath: /nfs
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: test-claim
```

這邊可以注意的是，當對應的 `PVC` 被創建之後，我們可以在 `NFS Server` 那邊觀察到，對應的 `/nfsshare` 資料夾裡面又被創建了一個新的資料夾 `default-test-claim-pvc-e0ae384c-0c45-11e9-be59-02e1e1d1e477`

這個含義就是該 `Deployment(Provisioner)` 針對該 `PVC` 的需求，動態的創建了一個資料夾，並且把該資料夾當作 `mount path` 給之後的 `Pod` 去使用。 所以使用不同 `PVC` 的 `Pod` 即使背後都是相同的 `NFS Server`, 也可以因為 `StorageClass` 動態規劃的幫忙而不會互相看到彼此的資料。


# Summary

最後用一個簡單的架構圖來進行一下比較
![](https://i.imgur.com/p2pxQhC.png)

原生的 `Kubernetes NFS` 的支援下，`NFS Server` 怎麼設定， 使用的 `NFS Client(Pod)` 就只能怎麼使用，掛載以及使用的資料夾比較固定。基本上要依賴 `NFS Server` 端的設計來滿足各種不同的使用情境，此外管理者必須要自己先行創立對應的 `PV/PVC` 供要使用的 `Pod` 來使用。

如果想要使用 `StorageClass` 這種動態分配空間的使用方式的話，可以參考一些孵化中的專案，`NFS-Client` 就是其中一個，透過自行實作 `Provisioner`，根據需求在整個 `NFS Server` 的目錄下創建更多更多的小目錄供對應的 `PVC` 去使用。
舉例來說，今天可以創建多個 PVC, 譬如 `DB-Data`, `Container-Log` 等不同的需求，然後 `Provisioner` 就會創建不同的資料夾供這些應用使用，最後每個 `Pod` 可以根據自己的需要來掛載特定的資料夾，而且互相彼此不會看到彼此資料。

唯一要注意的時，這邊也是有權限以及容量的問題，這部分就是 `NFS` 的基本限制。

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