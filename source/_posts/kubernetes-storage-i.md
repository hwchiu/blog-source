---
title: "Kubernetes X Storage (I)"
keywords: 'PV,PVC,StorageClass,Kubernetes, Compare'
abbrlink: 28691
date: 2018-11-10 12:47:55
tags:
  - Storage
  - Kubernetes
description: Kubernetes 的架構中，對於 Network/Storage/Device/Container 都有高度的抽象化，透過這些抽象化讓 Kubernetes 能夠更專心的發展自身平台與介面，讓更多的廠商與第三方專案專心開發發展自己的特色並且透過共通介面與 Kubernetes 協同合作。於儲存系統方面，最廣為人知的就是 PersistentVolume(PV), PersistentVolumeClaim(PVC) 以及 StorageClass, 這三者到底在 Kubernetes 叢集中扮演什麼樣的角色，彼此如何互相合作以及彼此各自有什麼樣的限制與應用，都會在本文中跟大家詳細介紹，希望能夠透過這篇文章讓大家更瞭解 PV/PVC/SC 的概念

---


# Preface

之前談的很多關於網路相關的部分，這次要來談談 Storage 儲存這一塊再容器世界中，也就是 Kubernetes 叢集的架構中，這些 Storage 扮演者什麼樣的地位，以及叢集的管理者以及容器的部屬者可以如何整合自己的容器系統與對應的 Storage 儲存系統。

我們都知道， Container 本身的特點就是移除快重啟也快，特別是搭配 Kubernetes Deployment/Replicaset 等資源調度可以確保容器能正確運行。
然而部分的容器應用場景下會有檔案相關的應用，此時若容器重啟是否能夠確保該檔案資料一致就是一個很大的問題。

舉例來說，今天有一個 MongoDB 的容器運行於 Kubernetes 之中，該 MongoDB 的容器會透過容器內的檔案系統創建一個資料夾來維護該資料庫的所有資訊。

今天若遇到了一些情境，譬如機器損毀，Rolling Upgrade 或是其他的情形導致該 MongoDB 的 Pod 需要重啟。在此情況下，新起來的 MongoDB 是否能夠繼續使用之前的資料? 抑或是每次都需要重新維護當前容器所需要的資料?

這個議題就是所謂的 Persistent Storage, 所謂的持久化儲存系統，再上述的範例中我相信大部分的人都會需要能夠讓相關的容器使用相同的資料來進行存取，因此接下來就會跟大家分享一下再 Kubernetes 的架構中，這方面相關的概念，特別是 `PersistentVolume`, `PersistentVolumeClame` 以及 `StorageClass` 這三個常見的用法。


# Overview

首先，我們先用下列的圖示來看一下整體的架構，這個架構包含了從最上層 Pod 以及到最底層的 Storage Provider.

![Imgur](https://i.imgur.com/btUPrUK.png)

基本上，我們先忽略 `StorageClass`，只需要有 `PersistentVolume` 以及 `PersistentVolumeClaim` 就可以滿足我們的需求，而 `StorageClass` 只是讓這一切變得更加簡單與方便。

設想一下，如果今天你是一個 `Kubernetes` 架構的設計者，為了解決這個問題，你會怎麼去設計怎麼儲存相關的整體架構?
該架構中，可能會希望
1. 可以支援不同的儲存方式，譬如 Block Devie, FileSystem, 同時 FileSystem 又可以有不同的支援，譬如 NFS,Ceph,GlusterFS。同時也希望能夠支援 Public Cloud
2. 容器(Pod)可以去存取該儲存系統，透過某種映射方式，希望可以在相同的`key`的情況下拿到相同的資料。這樣就可以確保我們的容器 (Pod) 可以再重啟後繼續存取相同的資料，同時不同的 Pod 也可以有機會存取到相同的資料，只要大家都使用相同的 `Key`
3. 可以根據不同的需求來使用儲存空間，譬如空間大小，存取模式，效能高低，快照等眾多特性

首先為了滿足 **(1)** 的需求，會需要有一層抽象層來處理，其背後銜接各種儲存系統的實作，並且提供相關的儲存使用方法供上層使用
同時，為了讓最上層的 `Pod` 再使用上可以不需要考慮底層儲存系統的細節，這邊會必須要使用更簡單且統一的格式來描述該儲存系統，因此這邊也需要一個抽象層的處理

將這兩個概念結合起來，其實就是所謂的 `PersistentVolume` 以及 `PersistentVolumeClaim` 這兩個代表性的資源。

## PersistentVolume(PV)
`PersistentVolume(PV)` 代表的就是背後儲存系統的叢集資源，因此在每個 `PV` 的描述中都要去描述該 `PV` 最後要銜接的儲存系統的相關資訊，譬如如果是 `NFS` 的話，就需要去描述相關的 `Server IP` 以及 `ExportPath` 等 `NFS` 相關的資訊。

`PV` 作為一個與儲存系統銜接的抽象層，除了包含儲存相關的參數外，也要提供一些相同的參數供上層使用。 `PV` 的實現則要根據這些參數嘗試去跟後端的儲存系統要求配置符合相關需求的儲存空間供容器使用。

接下來看一下對於一個代表儲存系統於叢集內資源的代表，其本身有哪些相關的參數可以使用

### VolumeMode
首先是一個我覺得大家比較少注意也比較不會去使用的參數，所謂的 `VolumeMode` 代表的是到底是 `FileSystem` 還是 `Block Device`.
一般我們在使用的都是已經格式化成為 `FileSystem` 的類型，譬如 `ext4`,`btrfs`,`zfs`. 然而有些應用會想要直接使用 `BlockDevice` 來使用，譬如使用 `/dev/sda5` 之類的方式。
預設的情況下都還是 `FileSystem`，因此沒有特別需求的話這個參數不太需要處理。

### Capacity
理想的狀況下，為了有效地控制與管理儲存空間，能夠事先規劃所需的容量大小並且請求相對應的儲存空間來使用是個比較好的應用方式。
於此前提下， `Kubernetes` 提供了透過 `Capacity` 的方式來表明這個 `PV` 希望儲存後端所提供的儲存空間至少要滿足這對應的大小.

以目前規劃的藍圖來說，除了單純的儲存空間大小外，未來也希望能夠規劃以效能為考慮的 `IOPS (Input/Otput Per Secsion)`, 詳細的可以參閱
[Kubernetes Resource Model](
https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md)

此外，要特別注意的是，並不是所有的儲存空間都能夠依照這個需求來滿足對應的條件，譬如以 `NFS` 本身其實就不支援這個選項。(不支援的意思是可以設定，但是實作上不會採用)

### AccessMode
`AccessMode` 則是設定使用端可以如何使用這個儲存空間，主要面向的對象是 `讀寫` 的設定，以及是否可以同時多個節點進行`讀寫`。


設定分成三種
1. ReadWriteOnce
請求到的該塊空間只能同時給一個節點使用，節點上的各種容器使用可以同時
進行`讀取`或是`寫入`的動作。
2. ReadOnlyMany
請求到的該塊空間可以同時給多個節點使用，但是節點上的各種容器使用都只能基於`讀取`這種沒有寫入的操作。
3. ReadWrtieMany
請求到的該塊空間可以同時給多個節點使用，且大家要進行讀取或是寫入等動作都是沒有問題的。

這邊比較有趣的是所有的同步單位都是基於`節點`而非基於`容器`，我個人的理解是因為 `Kubernetes` 底層的實作會是在節點上根據需求產生一個對應的儲存空間，接者該儲存空間則是會透過容器本身的映射方式，掛載到多個容器內去進行寫入存取。

此外，如果使用情境是如 `ReadWriteMany` 所敘述，該儲存空間可以再多個節點上同時進行讀寫，要如何確保檔案內容的一致性?
這邊的則是對應的儲存系統背後要自己負責處理，`Kubernetes` 本身不涉及任何儲存系統的實現與保護。
因此能夠支援 `ReadWriteMany` 的儲存空間其實並不多，如 `NFS` 或是一些分散式檔案系統 `Ceph/GlusterFS`.

### Reclaim Policy
對於 `PV` 來說，`Reclaim Policy` 的含義則是當使用自己的`PVC`被刪除時，這個`PV`要如何去處理要來的儲存空間。

舉例來說，假如我希望這塊儲存空間的資料可以重複被利用，我希望這些資料的刪減都是由管理員或是使用者自行處理，則我們應該要將該 `Policy` 設定成 `Retain`.

然而其他的情況下，可能會希望當 `PVC` 被移除後，就將對應的 `PV` 一併刪除，直接將該儲存空間移除，因為沒有需要重複利用的情境。
那就可以將該 `Policy` 宣告為 `Delete`

使用上還是根據使用情境來設定，沒有一定的對錯與最好的選擇

### Class
Class 類似標籤的概念，這邊先談簡單的概念，可以透過對 `PV` 設定該 `Class`(實際上在 Yaml 中叫做 `StorageClassName`）來限制只有含有相同標籤的 `PVC` 可以使用。


譬如我們可以針對這些 `PV` 設定不同的標籤，有可能是 `SSD` 或是慢速的`HDD`, 然後最上層的 `PVC` 可以再根據需求與標籤來從特定的 `PV` 中選出一個符合規則的儲存空間供上層使用。

實際上 `Class` 對於 `StorageClass` 那邊還有別的用途，這個部分等到後面再來詳談。

### Claim
這是一個在官方文件中沒有提到的參數，但是某些情況下還滿好用的，可以透過該參數去描述我只有特定的 `PVC` 能夠使用該 `PV`, 這部分實際上是透過 `PVC` 的 `namespace` 以及 `meta name` 來進行一個映射，並且透過這個方式可以將特定的 `PV` 與 `PVC` 給綁定。

詳細的用法可以參閱 [can-a-pvc-be-bound-to-a-specific-pv](https://stackoverflow.com/questions/34282704/can-a-pvc-be-bound-to-a-specific-pv)



### Mount Options
上述的所有選項都比較偏向儲存空間的能力與特性，而這個 `Mount Options` 則是描述當 `Kubernetes` 叢集中的節點想要掛載這個目標的儲存空間時，是否有一些額外的參數可以匹配與使用.

關於 `Mount Options` 的討論與構想，可以直接參閱 [Support Volume Mount Options #168
](https://github.com/kubernetes/enhancements/issues/168)


### Others
看了這麼多參數，那到底要如何去描述背後的儲存空間設定?
實際上在 `Yaml` 中並沒有一個固定的 `key` 去描述到底該如何使用背後的儲存空間。反而是每個儲存設定都會有一個屬於自己的 `key`，然後再該 `key` 底下去描述對應的資源與參數。

舉例來說使用 `NFS` 跟使用 `GlusterFS` 則會完全不同，可以參考下列兩個 `Yaml` 檔案
```yaml=
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

```yaml=
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gluster-default-volume
  annotations:
    pv.beta.kubernetes.io/gid: "590"
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  glusterfs:
    endpoints: glusterfs-cluster
    path: myVol1
    readOnly: false
  persistentVolumeReclaimPolicy: Retain
```

這邊的原因主要是因為 `kubernetes` 後端的實現(`golang`) 在進行物件的解析的時候，是採用下列的結構, 有興趣的人可以研究一下其背後實現原理的。

```golang=
type PersistentVolumeSource struct {
	// GCEPersistentDisk represents a GCE Disk resource that is attached to a
	// kubelet's host machine and then exposed to the pod. Provisioned by an admin.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#gcepersistentdisk
	// +optional
	GCEPersistentDisk *GCEPersistentDiskVolumeSource `json:"gcePersistentDisk,omitempty" protobuf:"bytes,1,opt,name=gcePersistentDisk"`
	// AWSElasticBlockStore represents an AWS Disk resource that is attached to a
	// kubelet's host machine and then exposed to the pod.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#awselasticblockstore
	// +optional
	AWSElasticBlockStore *AWSElasticBlockStoreVolumeSource `json:"awsElasticBlockStore,omitempty" protobuf:"bytes,2,opt,name=awsElasticBlockStore"`
	// HostPath represents a directory on the host.
	// Provisioned by a developer or tester.
	// This is useful for single-node development and testing only!
	// On-host storage is not supported in any way and WILL NOT WORK in a multi-node cluster.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#hostpath
	// +optional
	HostPath *HostPathVolumeSource `json:"hostPath,omitempty" protobuf:"bytes,3,opt,name=hostPath"`
	// Glusterfs represents a Glusterfs volume that is attached to a host and
	// exposed to the pod. Provisioned by an admin.
	// More info: https://releases.k8s.io/HEAD/examples/volumes/glusterfs/README.md
	// +optional
	Glusterfs *GlusterfsPersistentVolumeSource `json:"glusterfs,omitempty" protobuf:"bytes,4,opt,name=glusterfs"`
	// NFS represents an NFS mount on the host. Provisioned by an admin.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#nfs
	// +optional
	NFS *NFSVolumeSource `json:"nfs,omitempty" protobuf:"bytes,5,opt,name=nfs"`
	// RBD represents a Rados Block Device mount on the host that shares a pod's lifetime.
	// More info: https://releases.k8s.io/HEAD/examples/volumes/rbd/README.md
	// +optional
	RBD *RBDPersistentVolumeSource `json:"rbd,omitempty" protobuf:"bytes,6,opt,name=rbd"`
	// ISCSI represents an ISCSI Disk resource that is attached to a
	// kubelet's host machine and then exposed to the pod. Provisioned by an admin.
	// +optional
	ISCSI *ISCSIPersistentVolumeSource `json:"iscsi,omitempty" protobuf:"bytes,7,opt,name=iscsi"`
	// Cinder represents a cinder volume attached and mounted on kubelets host machine
	// More info: https://releases.k8s.io/HEAD/examples/mysql-cinder-pd/README.md
	// +optional
	Cinder *CinderPersistentVolumeSource `json:"cinder,omitempty" protobuf:"bytes,8,opt,name=cinder"`
	// CephFS represents a Ceph FS mount on the host that shares a pod's lifetime
	// +optional
	CephFS *CephFSPersistentVolumeSource `json:"cephfs,omitempty" protobuf:"bytes,9,opt,name=cephfs"`
	// FC represents a Fibre Channel resource that is attached to a kubelet's host machine and then exposed to the pod.
	// +optional
	FC *FCVolumeSource `json:"fc,omitempty" protobuf:"bytes,10,opt,name=fc"`
	// Flocker represents a Flocker volume attached to a kubelet's host machine and exposed to the pod for its usage. This depends on the Flocker control service being running
	// +optional
	Flocker *FlockerVolumeSource `json:"flocker,omitempty" protobuf:"bytes,11,opt,name=flocker"`
	// FlexVolume represents a generic volume resource that is
	// provisioned/attached using an exec based plugin.
	// +optional
	FlexVolume *FlexPersistentVolumeSource `json:"flexVolume,omitempty" protobuf:"bytes,12,opt,name=flexVolume"`
	// AzureFile represents an Azure File Service mount on the host and bind mount to the pod.
	// +optional
	AzureFile *AzureFilePersistentVolumeSource `json:"azureFile,omitempty" protobuf:"bytes,13,opt,name=azureFile"`
	// VsphereVolume represents a vSphere volume attached and mounted on kubelets host machine
	// +optional
	VsphereVolume *VsphereVirtualDiskVolumeSource `json:"vsphereVolume,omitempty" protobuf:"bytes,14,opt,name=vsphereVolume"`
	// Quobyte represents a Quobyte mount on the host that shares a pod's lifetime
	// +optional
	Quobyte *QuobyteVolumeSource `json:"quobyte,omitempty" protobuf:"bytes,15,opt,name=quobyte"`
	// AzureDisk represents an Azure Data Disk mount on the host and bind mount to the pod.
	// +optional
	AzureDisk *AzureDiskVolumeSource `json:"azureDisk,omitempty" protobuf:"bytes,16,opt,name=azureDisk"`
	// PhotonPersistentDisk represents a PhotonController persistent disk attached and mounted on kubelets host machine
	PhotonPersistentDisk *PhotonPersistentDiskVolumeSource `json:"photonPersistentDisk,omitempty" protobuf:"bytes,17,opt,name=photonPersistentDisk"`
	// PortworxVolume represents a portworx volume attached and mounted on kubelets host machine
	// +optional
	PortworxVolume *PortworxVolumeSource `json:"portworxVolume,omitempty" protobuf:"bytes,18,opt,name=portworxVolume"`
	// ScaleIO represents a ScaleIO persistent volume attached and mounted on Kubernetes nodes.
	// +optional
	ScaleIO *ScaleIOPersistentVolumeSource `json:"scaleIO,omitempty" protobuf:"bytes,19,opt,name=scaleIO"`
	// Local represents directly-attached storage with node affinity
	// +optional
	Local *LocalVolumeSource `json:"local,omitempty" protobuf:"bytes,20,opt,name=local"`
	// StorageOS represents a StorageOS volume that is attached to the kubelet's host machine and mounted into the pod
	// More info: https://releases.k8s.io/HEAD/examples/volumes/storageos/README.md
	// +optional
	StorageOS *StorageOSPersistentVolumeSource `json:"storageos,omitempty" protobuf:"bytes,21,opt,name=storageos"`
	// CSI represents storage that handled by an external CSI driver (Beta feature).
	// +optional
	CSI *CSIPersistentVolumeSource `json:"csi,omitempty" protobuf:"bytes,22,opt,name=csi"`
}
```

## PersistentVolumeClaim(PVC)
`PersistentVolume` 是與後端儲存空間連接的叢集資源，而 `PersistentVolumeClaim(PVC)` 則是銜接 `Pod` 與 `PV` 的中介抽象層，可以說是容器本身對於儲存需求的資源請求。

使用上，`Pod` 本身會去描述想要使用哪個 `PVC`，並且希望把該 `PVC` 掛載到 `Pod` 內使用。
而 `PVC` 本身則是會根據自身的需求，去**找尋是否有符合的PV**可以使用，並且將其該 `PVC` 與對應的 `PV` 進行一個 `Binding` 的動作。

### Resourecs
如同上面所述， `PV` 所描述的是叢集資源，而`PVC`描述的則是資源請求，在參數方面基本上 `PV` 跟 `PVC` 都會有相同的描述。不論是 `Capacity`, `AccessMode`, `VolumeMode`.

因此 `PVC` 只會去找尋能夠符合其條件的 `PV` 來進行挑選的動作。


### Binding
`Binding` 這邊拉出來特別解釋一下，實際上 `PV` 與 `PVC` 的概念
`PV` 代表的是從後端儲存設備根據資源需求所取得的儲存空間。
`PVC` 則是對於希望從`眾多的PV`中選出一個符合需求的 `PV`來使用，這個挑選的過程就稱為 `Binding`.

舉例來說，今天系統中創建了三個 `PV`, 每個 `PV` 都宣稱自己有 `50Gb` 的空間可以使用。這時候有個 `PVC` 的條件是希望找到一個至少有 `20Gb` 空間的 `PV` 來使用。
那實際上這三個 `PV` 都可以被使用，至於最後是哪個`PV`被選到，我這邊沒看到太多詳細且可證實的說法，之後若有看到會再補充。

如果今天有特別的需求，希望特定的 `PVC` 可以挑選到特定的 `PV`, 這邊可以透過前述講過的 `Claim` 參數來達成，只要於 `PV` 的設定中去描述對應 `Claim` 的 `namespace/name`, 並且與目標的 `PVC` 中去使用這些數值，就可以確保該 `PVC` 一定會用到該 `PV` 了。


## PV/PVC
我們重新再檢視一次剛剛看過的圖表
![Imgur](https://i.imgur.com/btUPrUK.png)

透過前述對於 `PV` 與 `PVC` 概念的闡述，我們現在可以知道
1. Pod 裡面會描述 `Volume`, 而 `Volume` 則是會描述對應的 `PVC` 名稱
2. `PVC` 實際上會根據自己的資源去系統中找到符合需求的叢集儲存資源`PV`, 並且將這兩者給 `Binding` 起來
3. `PV` 則是會負責跟對應的後端儲存設備進行連接，詳細的描述要如何使用後端儲存資源

將這個工作流程用實際上操作的用法來看，系統管理員與叢集使用者要如何協同合作來使用呢?
舉一個簡單的範例來說
1. 首先，系統管理員必須要先根據系統情境與架構創建好相對應的 `PV`
2. 使用者根據自己的需求，去系統上創建對應的 `PVC`, 並且期望該 `PVC` 能夠找到符合需求的 `PV` 來使用
3. 最後使用者部署相關的 `Pod` 於容器中時，就能夠將該 `PVC` 給掛載到容器中使用

這個流程中聽起來順暢但是實際上卻有一個用起來不夠方便的地方，就在於系統管理員必須要先創立相關的 `PV` 來使用，而且對於系統管理員來說，他可能並不知道使用者要的 `PVC` 資源到底是什麼，因此並沒有很好的規劃剛剛好的空間來使用。

我們將這種需要系統管理員事先設定 `PV` 且讓 `PVC` 根據需求自己去尋找 `PV` 的類型稱為 `Static`。

而相對於 `Static`，就會有所謂的 `Dynamic` 的概念出現，因此接下來就是要來介紹 `StorageClass` 的使用了。

## StorageClass
`Dynamic` 的概念最大的優勢就是，系統管理員不需要事先設定 `PV`, 而 `PV` 的創建都是根據需求創造出來的，這有兩個好處
1. 系統管理員不需要事先設定相關的 `PV`
2. 創建出來的 `PV` 都能夠完全符合 `PVC` 真正的資源需求，不會有浪費的情形

為了滿足這個想法，於是我們要來使用 `StorageClass` 了，對於 `StorageClass` 來說，我們先來觀看一下整體的使用流程。

1. 系統管理員根據需求與架構，創建對應的 `StorageClas` 物件，該物件也需要描述後端的儲存空間，但是不需要描述相對應的資源大小
2. 使用者接下來根據需求創建 `PVC`, 而這些 `PVC`內描述我要使用特定的 `StorageClass`
3. `kubernetes` 觀察到有 `PVC` 想要使用 `StorageClass` 且發現該 `StorageClass` 物件存在，就會透過 `StorageClass` 的屬性以及 `PVC` 的需求創造一個完全符合該 `PVC` 需求的 `PV`
4. 最後該 `PVC` 與該 `PV` 就可以綁定並且供 `Pod` 使用


接下來細看一下 `StorageClass` 的參數以及一些概念

### Provisioner
由於現在 `PV` 的創建是自動的，但是因為 `PV` 本身就跟後端的儲存空間有直接的連接關係，因此我們在 `StorageClass` 也要有能力去描述後端的儲存空間資訊。

所以實際上在使用的時候必須要在該 `StorageClass` 的`Yaml` 中透過 `Provisioner` 去描述該 `StorageClass` 背後使用的儲存系統是哪一套解決方案。

然而要注意的是，並不是所有 `PV` 目前支援的儲存系統都有對應的 `StorageClass Provisioner` 支援。

支援的部分除了官方自己已經合併到主線以外，也有第三方的解決方案再擴充相關的 `StorageClass Provisioner`.

詳細的支援列表可以參考 [Kubernetes StorageClass Provisioner](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)

### Parameters
前面都有看到 `PV` 內對於每個儲存欄位都有自己的一個結構去描述相關的參數，而在 `StorageClass` 內實際上並沒有這種格式來使用，然而不同的儲存系統可能也要有不同的相關參數來使用，因此在 `StorageClass` 這邊則是採用了一個固定的格式 `Parameters` ，裡面則是各式各樣的 `key:value` 的形式， `kubernetes` 會將這些資訊全部傳送到 `Provisioner`，讓 `Provisioner` 自行解讀來處理。

以下是一個 `StorageClass` 的範例
```yaml=
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gluster-vol-default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

### Binding
之前有提到過，`PVC` 在找尋 `PV` 的時候會尋找資源符合的來使用，但是如果使用的是透過 `StorageClass` 來幫你創建的 `PV` 的話，實際上系統會幫忙再 `PV` 的描述中使用 `Claim` 的欄位，確保你的 `PVC` 與創建出來的 `PV` 可以完全地符合，且也只會使用這個 `PV`.

此外，透過 `StorageClass` 所創建出來的 `PV`，其 `StorageClassName` 的欄位就會自動被捕上該 `StorageClass` 的名稱，對於資訊的補充也更加完整。

所以就 `PV/PVC` 裡面的 `StorageClass` 的用法，若系統上沒有 `StorageClass` 的資源的話，可以單純用來限制 `PVC` 與 `PV` 尋找的範圍。若有 `StorageClass` 的話，就可以讓 `StorageClass` 來幫忙創建與維護相關的 `PV`.

#
最後，重新檢視一次最初的那張圖片
![Imgur](https://i.imgur.com/btUPrUK.png)

左邊是最基本的用法，自行創建 `PV` 與 `PVC`, 並且將所需要的資源與參數都填寫完畢後，由容器本身去選擇要使用哪個 `PVC`.

右邊則是套用上 `Storage Class` 這種動態創建 `PV` 的用法，將 `PV` 的創造與維護 **(PV的刪除也是依賴 Reclaim Policy)** 讓 `kubernetes` 本身幫忙維護。 而創建出來的 `PV` 所擁有的系統資源以及存取模式等都會跟 `PVC` 完全一致，確保不會有額外的系統資源浪費。

下一張會跟大家分享一下 `NFS` 這個儲存設備到底在 `Kubernetes` 裡面可以怎麼使用，除了從最基本的 `PV/PVC` 一般常見的用法外，也會跟大家分享一下如何使用第三方的 `NFS Provisioner` 來透過 `StorageClass` 使用 `NFS` 做為容器的儲存空間

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/mLwKb3zrx

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4XNG

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