---
title: 常見 CNI (Container Network Interface) Plugin 介紹
keywords: 'CNI, compare, calico,falnnel,multus,introduction, kubernetes cni'
abbrlink: 9548
date: 2018-10-28 07:54:27
tags:
  - Network
  - Docker
  - Kubernetes
  - CNI
description: 作為一個 Kubernetes 使用者，可能都有聽過 CNI/CRI/CSI 等眾多的介面。而 CNI 作為一個掌管整個 Kubernetes 叢集網路的核心元件，負責提供各式各樣的網路功能。目前為數眾多的開源 CNI 專案們，到底各別擁有什麼樣的特性與效果，作為一個管理者在選擇 CNI 的時候應該怎麼考慮。本文針對常見的 CNI 專案們進行了一個簡單的分析與介紹，讓讀者能夠更加得清楚自己需要的功能及應用是什麼，才能夠更聰明的選擇所要的 CNI

---


# Preface 
本篇文章要來幫大家導讀一下目前常見的 `CNI Plugin` 有哪些，大致上的內容主要會含括這個 `Plugin` 的基本功能以及主打的特色，可能使用的場景與情境。
老話一句，基本上叢集管理員還是要`明確的清楚自己的需求`,不要盲目的追求任何一套 `CNI Plugin`, 沒有存在一套世界最強的 `CNI Plugin` 可以滿足所有的使用情境與需求。


# Basic CNI
一開始先跟大家介紹一些大家應該都沒聽過的 `CNI Plugin`
1. bridge
2. host-device
3. ipvlan
4. macvlan
5. ptp
6. vlan
7. loopback

這些 `CNI Plugin` 都是由 `CNI` 專案所提供的基本功能，其目的非常簡單，就是提供最簡單且單一的網路架構。

以 `Bridge` 為範例，就是一個透過 `Linux Bridge` 將容器與宿主機給串連起來的方式，除此之外就沒有任何功能了。

而 `ptp` 就是一個簡單的 `Point to Point` 的架構，中間甚至連虛擬交換機都沒有，單純的透過 `veth` 將容器與宿主機連接再一起。

但是 `CNI` 迷人的地方就在於這些功能的整合，每個 `CNI` 專注於一個簡單的功能，彼此又互相整合來完成各式各樣的強大功能。

有興趣的話可以參閱一下 [containernetworking/plugins](https://github.com/containernetworking/plugins/tree/master/plugins/main)


# Flannel
`Flannel` 我覺得應該是 `CNI Plugin` 裡面頂級出名的，幾乎各種教學文件都可以看到使用 `Flannel` 作為基本的 `CNI Pugin`. 其實從 `Flannel` 的功能面以及安裝性來說，我認為 `Flannel` 非常簡單且已經可以滿足大部分安裝者的基本需求。

此外，`Flannel` 其實背後則是會呼叫 `Bridge` 這個 `CNI` 來幫忙建立基本的 `Veth/Bridge` 等功能，自己則是專注於 `Overlay Network` 的管理。

我們從兩個面向來看 `Flannel` 的特色

## Installation
`Flannel` 安裝非常簡單，實際上只需要部屬一個 `Yaml` 檔案即可。
該 `Yaml` 裡面其實會牽扯到不少元件，譬如 `DaemonSet`, `InitContainer` 等許多有趣的玩法，這邊就不再贅述，有興趣的可以想想看一個問題

你要如何讓任何加入到k8s的新節點都能夠自動安裝 CNI 需要的 binary 以及 config ?


## Features
`Flannel` 功能非常簡單，就是且只有 `Overlay Network`，目標是提供`Pod`與`Pod`之間跨節點的溝通，提供 Layer3 IPv4 的能力來處理節點與節點之間傳輸的辦法。
目前其採用的實作方式有
1. VXLAN
2. UDP 封裝
3. Host-GW 靜態路由設定

詳細的實作內容請參閱[GitHub Flannel](https://github.com/coreos/flannel/blob/master/Documentation/backends.md)

# Calico
另外一個也是非常知名的 `CNI Plugin` 就是 `Calico`, 相對於 `Flannel` 來說， `Calico` 提供的功能則相對的多，而這些功能與特色都能夠對應不同的網路環境。

`Calico` 專案的開發相對於 `Flannel` 來得更加活躍且也不停的有各種功能出現，畢竟相對於 `Flannel` 單純想提供`Pod x Pod` 連線功能來說， `Calico` 想提供的則是更多的功能，自然而然的發展就會比較廣且活躍。

## Installation
安裝方便基本上可以很簡單，如同 `Flannel` 一樣透過一個 `Yaml` 就可以安裝基本的 `CNI` 資源到 Kubernetes 叢集中。此外也可以很複雜到需要安裝非常多的東西來提供更進階的功能。 `這部分取決於你想要採用的功能`。

舉例來說，如何建立存放 `Calico` 各節點溝通的資料就有兩種
1. etcd
2. kubernetes API

詳細的安裝教學可以參閱[官網](https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/)

## Features
`Calico` 支援的平台非常多種，本身支援 `CNI` 介面使得支援 `CNI` 的平台都可以使用外，其也可以透過 `Neutron Plugin` 的方式安裝於 `OpenStack` 的環境中，算是我目前看到支援度非常豐富的網路功能解決方案

在其功能方面，基本上主要以 `kubernetes` 該平台能夠提供的功能來探討，可以分成兩個面向來看，分別是 `Policy` 以及 `Network`。

### Policy
1. Network Policy
實際上，`Kubernetes` 本身有定義 `Network Policy` 的介面，但是實際上卻沒有實現這種能夠用來限制 `Pod` 與 `Pod` 之間連線的功能。 官網明確的說明這部分的實現要依賴 `CNI Plugin` 來處理。
而 `Calico` 就有實現這種介面，因此可以直接使用 `Kubernetes Network Policy` 的規範去設定相對應的 `Yaml` 來實現基本的`ACL (Access Control List)` 

2. Application Layer Policy
相對於基本的 `Kubernetes Network Policy` 外， `Calico` 本身透過 `Kubernetes CustomResourceDefinetion(CRD)` 提供了另外一層新的 `GlobalNetworkPolicy`.
該介面提供了基於 `HTTP` 方法或是路徑為基準的 `ACL`，本身的實現是透過 `Service Mesh` 的方式來進行 `ACL` 的處理，因此本身會依賴 `lstio` 的系統。在安裝上會需要更多的操作以及設定的細節來開啟此功能。

### Network
1. Overlay Network.
`Calico` 對於網路連線方面也有提供類似 `Flannel` 的功能，透過 `VXLAN` 或是 `IP in IP` 這類型的協定來封裝連線封包已達到 `Overlay Network` 的需求
2. Native Routing
除了上述的 `Overlay Network` 之外， `Calico` 最主打的就是第二種網路，完完全全的 `Underlay Network`, 不依賴任何的封包封裝協定。
`Calico` 的想法很簡單，雖然 `Overlay Network` 很方便，但是實際上每次的封包封裝都會增加整體封包的複雜度，同時也會增加封包的處理時間。
因此 `Calico` 透過 `IPTables` 以及 `Routing Table` 的方式來處理 `Layer3`路由的問題，讓不同節點之間的 `Pod` 可以透過 `Routing Table` 的方式直接互相連間而不需要再包覆一層額外的標頭檔來處理封包。

這部分的原理牽扯到 `Layer3` 路由以及相關閘道的 `MAC address` 的原理，有興趣的可以參考官方說明
[The Calico Data Path: IP Routing and iptables
](https://docs.projectcalico.org/v3.3/reference/architecture/data-path)
3. Public IP Address Assignment
`Calico` 有額外自行開發一套 `IPAM` 來管理 `IP` 及分配，在此架構中所有的 `IP` 都會從 `IP Pool` 中去取得。

同時搭配 `BGP` 等動態路由協定的幫忙，所有的 `Pod` 都可以被賦予一個可被存取的 `Public IP` 地址，同時這些 `Pod` 在對外進行存取的時候也不需要進行 `NAT` 的轉換，可減少一次封包轉換所造成的效能耗損

# Canal
`Canal` 是一個比較特別的 `CNI Plugin`, 其目的在於整合 `Flannel` 以及 `Calico` 這兩個 `CNI Plugin`. 希望使用 `Calico` 的 `Network Policy` 功能以及 `Flannel` 的 `Overlay Network` 功能。

因此整體上並沒有什麼特別獨特的功能，不過隨者時間演進，目前 `Canal` 專案也有大變化。

從 `Canal` 於 [Github canal](https://github.com/projectcalico/canal) 官網上面可以看到該專案在[ Jan 27,2018](https://github.com/projectcalico/canal/commit/54b58775578f0e3a306d9f2706e4d9b829fb7562#diff-04c6e90faac2675aa89e2176d2eec7d8) 更新了說明來表示該專案已經停止開發，目前已經可以直接在 `Calico` 的官網直接看到 `Calico` 與 `Flannel` 的整合方法以及安裝方式，因此 `Canal` 這個獨立的 `CNI Plugin` 基本上已經沒有需要使用的必要了。


# Nuage
`Nuage` 是一個基於 `SDN` 概念開發的`CNI Plugin`，基底層使用了 `OpenvSwitch` 作為軟體交換機，同時使用了 `OpenFlow` 此協定來控制 `OpenvSwitch`.
此外，為了能夠更加聰明有智慧的去透過 `OpenFlow` 控制 `OpenvSwitch`, 也必須要有一個 `OpenFlow Controller VSC` 來作為一個中央控管的管理者。

## Feature
`Nugae` 本身在網路連線方面，也有提供基於 `VXLAN` 協定的 `Overlay Network` 的應用。這部分應該是直接利用 `OpenvSwitch` 本身提供的 `VXLAN` 功能來完成。
此外針對 `Nuage` 也有實現 `Network Policy`, 其原理就是透過 `Openflow` 的規則下發到所有的 `OpenvSwitch` 來達到 `Pod` 與 `Pod` 之間的傳輸規則隔離。

下圖是一個簡單的示意圖來解釋 `Network Policy` 的運作原理，大致上就是當使用者透過 `Client API` 去寫入 `Network Policy` 的規則後，這些規則最後會被相關的應用程式給解讀並且轉達給 `VSC, SDN Controller`，最後 `VSC` 會把該規則對應到的資訊透過 `OpenFlow` 的方式寫入到對應的節點上面的 `OpenvSwitch` 上來達到 ACL 的功能。

![](https://3.bp.blogspot.com/-jJK65zh2wE8/WIE5o3HkXFI/AAAAAAAAA7U/QkoCoYnTWAEz60H0nyP4_wN0tVG3WVWAwCEw/s1600/k8spolicy.png)

# Ciliunm
`Ciliunm` 最厲害也是最主打的功能並不是網路方面的連接，反而是安全性以及平衡附載特點。
在其實作中採用了 `eBPF` 以及 `XDP` 這兩種這幾年逐漸受到重視的技術來處理封包。連線的封包能夠在網卡收到但是尚未送到 `Kernel` 前就先行處理，因此有更早期且更快速的效率來處理封包。

接下來就針對其各個特色功能來介紹

## Networking
網路連線方面提供兩種功能，`Overlay Network` 以及 `Native Routing`.
1. Overlay Network
`Overlay Network` 本身也是依賴 `VXLAN` 以及 `Geneve` 這些已知的協定來處理，`Ciliunm` 直接使用 `Kernel` 相關的功能來達到這些封裝效果，因此其支援度以及支援性則是依賴於使用的 `Kernel` 版本。
2. Native Routing
這個比較偏向給進階使用者使用的，叢集管理者必須要知道這方面的概念同時也要確保相關的 `Routing Table` 是可以運行的，同時也可以使用 `OSPF/BGP` 等相關的應用程式來輔助。
詳細的可以參考[Ciliunm Concepts](https://cilium.readthedocs.io/en/v1.2/concepts/#direct-native-routing-mode)
3. Load Balacing
負載平衡方面，`Cilium` 則使用了 `BPF` 的架構並且透過 `Hashing` 的方式來幫忙決定該連線最後的目標節點，

## Network Policy
`Ciluum` 除了實現了 `Kubernetes Network Policy` 這種基於 `Layer4/Layer3` 的防火牆外，本身也有額外實現了 `Layer 7` 的防火牆。
在 `Layer7` 方面支援了下列的協定
- REST/HTTP
- gRPC
- Kafka
已 `Kafka` 來說，可以使用下列的 `Yaml` 來描述相關的防火牆規則
```Yaml=
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
description: "enable empire-hq to produce to empire-announce and deathstar-plans"
metadata:
  name: "rule1"
spec:
  endpointSelector:
    matchLabels:
      app: kafka
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: empire-hq
    toPorts:
    - ports:
      - port: "9092"
        protocol: TCP
      rules:
        kafka:
        - role: "produce"
          topic: "deathstar-plans"
        - role: "produce"
          topic: "empire-announce"
```
詳細更多的用法可以參閱其[官網](http://docs.cilium.io/en/stable/) 來學習

# OVN
`OVN`, `Open Virtual Network`  是由 `OpenvSwitch` 的開發公司, `Nicira` 開發的一套 `OpenvSwitch` 管理工具。
最初開發 `OVN` 的用途就是希望能夠透過一個類似 `SDN Controller` 概念的應用程式來管理整個網路叢集中的 `OpenvSwitch`, 並且方便且有效率透過管理這些 `OpenvSwitch` 來達到各式各樣的網路功能，譬如簡單的 ACL 到多租戶的虛擬網路

如今 `OVN` 也透過 `CNI` 的介面以及相關的 `Kubernetes API` 來整合到 `Kubernetes` 環境中。

從功能面來說, `OVN` 的主軸還是在於網路功能的提供，除了最基本的 `Overlay Network` 之外，也能夠透過 `OpenvSwitch` 的架構來實現
如 `kubernetes service(ClusterIP,NodePort)` 等功能。

從目前的發展來看也許還沒有很強大的力量可以吸引使用者轉換過去，不過若能夠透過 `OVN` 配上 `OpenvSwitch` 來完成 `Kubernetes` 內部所有的 `Networking` 功能，則有招一日我認為若能夠將所有的 `OpenvSwitch` 都整合 `DPDK` 此框架則有機會可以一舉將整個 `Kubernetes` 內部所有網路傳輸的效能直接提到到 10Gbps 以上(數字上限很依賴調整，但是 10Gbps 基本上絕對沒問題).

有興趣玩玩的可以參考 [官網](https://github.com/openvswitch/ovn-kubernetes)，裡面也有附設相關的 `Vagrant` 讓開發者使用看看。

# SR-IOV
相對於前述所有的 `CNI` 都提供一個全面性的網路功能，接下來的幾個 `CNI` 則是會跟網卡的硬體資訊有更強烈的整合。
`SR-IOV` 的介紹可以參考 [NFV 網路技術介紹 – SR-IOV](https://blog.pichuang.com.tw/nfv-sr-iov/) 這篇文章來學習。

比較簡單的理解就可以想成硬體網卡端會直接映射一個虛擬網卡到對應的 `Container` 內部去使用，這種情況下該 `Container` 內往該虛擬網卡送出去的封包都會直接從對外層的實體網卡送出去，不會經過宿主機內部的任何軟體交換機處理。

好處來說就是效能會更高，速度更快。壞處來說就是基本上 `Kubernetes` 原先透過 `IPTables` 或是 `IPvS` 提供的任何 `Kubernetes Networking Functions` (Services/DNS) 都會失效。

此外 `SR-IOV` 也有 `VF` 數量上的限制，所以使用上絕對不會是每個 `Pod` 都會使用，而是要依據自己的需求以及對應的情境來使用。

## Example
一個簡單的 `CNI` 設定檔範例如下
```yaml=
{
    "name": "mynet",
    "type": "sriov",
    "if0": "enp1s0f1",
    "ipam": {
        "type": "host-local",
        "subnet": "10.55.206.0/26",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ],
        "gateway": "10.55.206.1"
    }
}
```

使用上強烈建議要搭配 `Intel` 針對 `SR-IOV` 所開發的 `Device Plugin` 來處理 `VF` 的同步與管理，可以減少更多對硬體資訊的依賴性。

有興趣的可以參閱 `Intel` 所維護的 [SRIOV-CNI](https://github.com/intel/sriov-cni)

SR-IOV 最早期的 Plugin 並不是 Intel 所開發的，但是 Intel 將其 Fork 回來並且加入了更多的功能，同時也在同一個 `Plugin` 之中去支援 DPDK 的功能


# DPDK
與 `SR-IOV` 類似，都是針對特定用途且有網卡資訊依賴性的 `CNI` 插件，主旨都在提供一個更高速且更低傳輸延遲的網路環境。

`DPDK` 就是一個 `Kernel ByPass` 的應用程式架構，透過 `Polling` 的方式來盡可能的使用 `CPU` 去輪詢網卡來提升網卡的處理效率，在使用上則必須要指定對應的實體網卡資訊。
一旦網卡使用上了 `DPDK`, 所有未客製化的常見工具 (ifconfig, ip) 等都沒有辦法檢視到該網卡的資訊，因為該網卡已經從 `Kernel` 內消失了。

## Example
一個關於 `DPDK` 的 `CNI` 設定 `Yaml` 如下
```yaml=
{
    "name": "mynet",
    "type": "sriov",
    "if0": "enp1s0f1",
    "if0name": "net0",
    "dpdk": {
        "kernel_driver":"ixgbevf",
        "dpdk_driver":"igb_uio",
        "dpdk_tool":"/opt/dpdk/usertools/dpdk-devbind.py"
    }
}
```

`Intel` 將 `DPDK` 相關的功能一起放在 `SRIOV-CNI` 裡面，可以到[SRIOV-CNI Github](https://github.com/intel/sriov-cni)參考用法

# Bond-cni
`Bonding`, 或是 `Teaming` 都是非常類似概念的一種實作，本文主要針對 `Bonding` 來介紹。

歡迎參考 `Redhat` 的文件來學習 `Teaming` 以及 `Bonding` 的差異
[If You Like Bonding, You Will Love Teaming](https://rhelblog.redhat.com/2014/06/23/team-driver/)


能夠將多張網卡抽象成一張網卡來使用，這種情況下該網卡能夠提供下列的功能
1. Load-Balancing
能夠針對 `Layer2/Layer3/Layer4` 等不同的規範讓不同的連線最後透過不同的實體網卡傳輸，舉例來說兩張 1G 的網卡綁在一起後是有機會能夠提供 2Gbps 的傳輸數率
2. Fault-tolerance
3. Active-Backup
當底層網卡有任何故障的時候，最上層的應用程式可以不需要意會到這種情況且依然有辦法繼續傳送網路封包。

## Example
相關的 `CNI` 設定如下
```yaml=
{
	"name": "mynet",
	"type": "flannel",
	"delegate": {
		"type": "bond",
		"mode": "active-backup",
		"miimon": "100",
		"links": [
            {
				"name": "ens3f2"
			},
			{
				"name": "ens3f2d1"
			}
		]
	}
}
```

這個 `CNI` Plugin 也是由 `Intel` 所開發的，不得不說 `Intel` 對於 `On-premise` 環境中的 `Networking Solution` 下足了功夫，缺少什麼就自己實現什麼並且開源，最後將這些各式各樣的 `CNI Plugin` 統整成為自己的解決方案。

有興趣了解更多的可以參閱其[intel/bond-cni](https://github.com/intel/bond-cni)

# Multus/CNI-Genie/Knitter
前面講了非常多的 `CNI Plugin`，從全面性的網路功能到依賴特定網卡型號的用途都有，接下來要介紹的 `CNI Plugin` 可以說是一個非常特別類型的 `CNI` 用法。

舉 `Kubernetes` 為範例，對於每一個創建的 `Pod` 只會呼叫一次 `CNI` 來設定相關的網路功能，然而在某些場景應用中，會特別希望該 `Pod` 中有多個網路介面。我這邊使用下圖作為一個範例介紹

試想一個情境，有很多的應用程式容器化之後要運行在 `Kubernetes` 上，這些容器本身需要互相溝通協調，同時這些容器本身有非常強烈的網路效能需求，譬如 `High Throughput/Low Latency`.

在這些應用程式尚未容器化以前，其架構中常常會規劃成 `Control Network` 以及 `Data Network`. 這些應用程式透過 `Control Network` 來傳輸控制用的資料，而透過 `Data Network` 來傳輸真正的資料。

這些應用程式容器化遷移到 `Kubernetes` 之後，要如何在 `Kubernetes` 上滿足這些需求?

就算今天 `Kubernetes` 不用 `IPTables` 而改用 `IPvS` 的模式來處理相關的功能，整體傳輸的速度還是受限於 `Linux Kernel`而沒有辦法達到令人滿意的境界，勢必要另尋它路來處理。
![Imgur](https://i.imgur.com/39xuTqb.png)

這種需求的情況下，有一種特別的 `CNI` 就誕生了，這類型的 `CNI` 本身沒有提供任何的網路應用功能，而是作為一個呼叫者去銜接數個 `CNI Plugins` 來處理。
這邊借用 `Multus` 的一張圖片來說
![](https://github.com/intel/multus-cni/raw/master/doc/images/multus_cni_pod.png)

最終目的就是希望每個 `Container` 被創立的時候都可以執行多次的 `CNI Plugin`.

因此整個架構就會是 `Kubernetes` --> `Special CNI` -> `Other CNIs`.

現在提供這種需求的 `CNI` 也不少，由於目的相同，大部分都是單純用法不同，因此我將這些 `CNI` 都放在一起。

`Mutlus-CNI` 是由 `Intel` 所主導開發的，看到這邊再來回想一下前面所介紹的 `SRIOV/DPDK/Bonding` 等也是由 `Intel` 所維護的相關 `CNI Plugin`。

不難想像 `Intel` 想要做的是一個整體的網路解決方案，先透過個別的 `CNI` 提供獨特的功能，最後透過 `Multus` 的方式將這些 `CNI` 全部串接起來來滿足使用者的需求。

## Example
這邊舉一個 `Intel` 提供的範例，看看是如何透過 `Multus` 串接各式各樣的 `CNI` 來使用

```yaml=
{
    "name": "multus-demo-network",
    "type": "multus",
    "delegates": [
        {
                "type": "sriov",
                #part of sriov plugin conf
                "if0": "enp12s0f0",
                "ipam": {
                        "type": "host-local",
                        "subnet": "10.56.217.0/24",
                        "rangeStart": "10.56.217.131",
                        "rangeEnd": "10.56.217.190",
                        "routes": [
                                { "dst": "0.0.0.0/0" }
                        ],
                        "gateway": "10.56.217.1"
                }
        },
        {
                "type": "ptp",
                "ipam": {
                        "type": "host-local",
                        "subnet": "10.168.1.0/24",
                        "rangeStart": "10.168.1.11",
                        "rangeEnd": "10.168.1.20",
                        "routes": [
                                { "dst": "0.0.0.0/0" }
                        ],
                        "gateway": "10.168.1.1"
                }
        },
        {
                "type": "flannel",
                "delegate": {
                        "isDefaultGateway": true
                }
        }
    ]
}
```



# Summary
礙於人生苦短，時間有限，本篇文章沒有辦法將所有的 `CNI Plugin` 都介紹一遍，還有非常多的 `CNI Plugin` 沒有介紹到，譬如 `Knitter`, `vhost-CNI`, `DANM` 甚至是一些`Plublic Cloud Provider`所開源的 `CNI` 專案.

不過我相信對於大部分公有雲的使用者來說，其實使用公有雲本身提供的網路解決方案都能夠滿足大部分的需求，如果你有需要在 `On-Premise` 的架構下去設計這些 `Kubernetes` 平台並且需要符合特定的使用情境的話，那一定要好好的想清楚自己的需求，尋找一套合適的`CNI`來使用，真的不行的話非常推薦自己撰寫一套 `CNI` 出來客製化自己的需求。

# Reference
- [Scaling Kubernetes deployments with Policy-Based Networking](https://kubernetes.io/blog/2017/01/scaling-kubernetes-deployments-with-policy-base-networking/)
- [Cilium Network Policy](http://docs.cilium.io/en/stable/policy/language/#http)
- [CNI - the Container Network Interface
](https://github.com/containernetworking/cni)
