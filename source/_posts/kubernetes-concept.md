---
title: '你到底知不知道什麼是 Kubernetes?'
keywords: 'Kubernetes,101, rumor, 誤解'
abbrlink: 15679
date: 2019-02-02 07:07:28
tags:
  - kubernetes 
description: Kubernetes 的蓬勃發展以及其人氣帶來廣泛地使用，然而就現實中，其實出現了不少關於 kubernetes 不太正確的想像與理解，愈來愈多的人因應 kubernetes 的發展就將 kubernetes 視為一個完美的解藥，能夠解決所有營運部署的所有困難與需求。最後發現現實與理想沒有辦法妥協時就會露出失望與無奈的表情。其實問題就出在一開始沒有理解到底 kubernetes 能夠帶來什麼樣的優勢以及本身有什麼樣的能力與限制。本文針對一些常見的三大資源，儲存/網路/運算 介紹了一下筆者自己觀察以及理解的概念去描述到底 kubernetes 能夠做什麼，不能夠做什麼

---

# Preface
2018年可以說是 Kubernetes 的一年，其聲勢浩蕩在全球捲起了一股停不下來的風潮，` Kubernetes` 可以說是一個無人不知無人不曉的技術，就算沒有使用過也一定有聽過 `Kubernetes` 這個詞，甚至是 `k8s` 這個縮寫。

然而 `人云亦云，以訛傳訛` 的事情也發生在 `Kubernetes` 身上，這年來我自己就聽到愈來愈多關於 `Kubernetes` 的誤解，譬如下列情況
A: 聽說 `Kubernetes` 很有名,網路功能特別強，我們整個專案都搬過去吧
B: 聽說 `Kubernetes` 很有名,儲存空間什麼都可以解決，我們直接上這個架構吧
C: 聽說 `Kubernetes` 很有名,容器配上`Docker` 非常厲害，我們直接上這個架構吧
D: 聽說 `Kubernetes` 很有名,什麼都可以做，我們取代 `Openstack` 吧

搞得最後 `Kubernetes` 變得跟`神`一樣，沒有解決不了的問題，凡事只要容器化，一切都解決了。

然而通常 `A/B/C/D` 都不是底下的工程人員，所以最後都會變成`用嘴巴生架構，用嘴巴解決問題`，底下實際的工程人員則是各種崩潰

{% note info %}
必須老實說，上述的情形很容易出現在某些體系的案子內，沒有根據實際需求而盲目追求風潮的下場只能說....
{% endnote %}

然而這些問題其實只要仔細的去思考下列問題，大部分都可以避免
1. 什麼是 Kubernetes
2. Kubernetes 能帶來什麼好處
3. 我為什麼要使用 Kubernetes

然而有趣的是，通常會講出上面謬誤話語的人通常也沒有很仔細的思考過這些問題, 所以這些聽起來覺得不合邏輯的討論與說法才會一而再再而三地出現。

所以這次特別想根據自身的理解與經驗，用這篇文章跟大家分享，到底 `Kubernetes` 在 `儲存`/`網路`/`計算` 方面能夠做什麼，不能夠做什麼。

這篇文章本質上不是技術文，不會去探討 `Kubernetes` 底層的太多的技術細節，反而會更偏向以概念的方式來探討到底 `Kubernetes` 本身到底能做什麼，不能做什麼。

# Storage(儲存)
## Before Kubernetes
Storage(儲存) 實際上一直都不是一個簡單處理的問題，從軟體面來看實際上牽扯到非常多的層級，譬如 Linux Kernel, FileSystem, Block/File-Level, Cache, Snapshot, Object Storage 等各式各樣的議題可以討論。
以檔案系統來說，光一個 `EXT4/BTRFS` 兩個檔案系統就有不少的評比與比較，何況是加上了 Distributed FileSystem(分散式作業系統)，譬如 `Ceph`, `GlusterFS` 等相關的解決方案進來後，一切事情又變得更加複雜。

此外還可以考慮到其他的軟體相關儲存技術，譬如 `RAID`, `LVM`, 甚至是各式各樣的`Read/Write Cache` 及`DRBD` 各種不同取向的解決方案，都會因為 **使用者的需求而有不一樣的選擇**.

異地備援，容錯機制，快照，重複資料刪除等超多相關的議題基本上從來沒有一個完美的解法能夠滿足所有使用情境。

NetApp, Nutanix, 家用/企業 NAS 等眾多廠商專注於儲存解決方案的提供，從單一機器的擴充到超融合架構(HCI)都是服務的對象之一

光這樣看下來就知道儲存技術真的不簡單，`Kubernetes` 何德何能可以以一個 `Container Orchestrator` 平台來解決所有事情?

舉一個最簡單的範例來說， `NFS(Network File System)` 是一個普遍都聽過也滿常被使用的儲存方案，這種 `Client/Server` 架構下，系統管理者針對 `NFS Server` 進行設定跟擴充，基本上`NFS Client` 大部分都不知道，甚至沒有感覺 (**Mount options 除外**)

譬如除了基本的檔案存取外，可以藉由 `RAID` 來提供基本硬碟容錯的功能，管理者可能會直接在 `NFS Server` 上進行 `MDADM` 來設定相關的 `Block Device` 並且基於上面提供 `Export` 供 `NFS` 使用，甚至底層套用不同的檔案系統 (EXT4/BTF4) 來獲取不同的功能與效能。
這類型的修改都是在 `NFS Server` 端去完成的，而 `NFS Client` 端不知道也無權責去進行這些功能的強化。

而 Kubernetes 就只是 `NFS Client` 的角色，所以整體背後的 `NFS Server` 能夠提供什麼樣的儲存功能與安全保障，對於 `Kubernetes` 來說已經超出其權責之外了，其能夠做的就是向 `NFS Server` 進行連線取得一個可以存取的網路位置罷了。

## Kubernetes
對於 `Kubernetes` 的定位來看，本身平台更專注地在於`介面標準`的制定與支援，在 `Kubernetes` 中該`介面`則是所謂的 `CSI(Container Storage Interface)`。
`CSI` 本身作為 `Kubernetes` 與 `Storage Solution` 的中介層。
`Kubernetes` 這邊專注於本身的元件，`PV/PVC/StorageClass` 這些元件作為中介層，往上銜接 `Pod`　等實際應用情形，往下則透過 `CSI` 與各式各樣的 `Storage Solution Provider` 銜接.

詳細的用法跟概念可以參閱 [Kubernetes Storage 101](https://www.hwchiu.com/kubernetes-storage-i.html)

若想了解更多 `CSI` 的設計原理跟組成，可以直接參閱可以參閱 [官方 Github Container-Storage-Interface](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)

## Summary
1. Kubernetes 本身不提供任何儲存功能, 透過標準介面 (CSI) 存取儲存伺服器
2. Kubernetes 本身也不去管什麼 `RAID`, `快照`, `分散式儲存`, `資料同步`, 這些都是後端儲存伺服器自行完成
3. 請針對自己的需求以及認知，選擇一個適合自己的儲存方案來使用
4. 不要認為 Kubernetes 可以幫你處理一切事情，沒有這麼強大也不應該這麼強大，請認份的學習儲存方面的概念與知識，然後與 Kubernetes 整合.
5. 遇到任何問題，可能是 Kubernetes 使用上的問題，也有可能是儲存伺服器本身的問題，這部分要仰賴管理者的經驗來處理

# Network(網路)
網路這個議題也非常有趣，我認為談到 `Kubernetes` 與 `Networking` 的關係時，可以有兩個方向去探討
1. 如何提供網路功能給 `Kubernetes` 內運行的容器
2. 如何將 `Kubernetes` 應用到網路服務提供商

事實上，大部分(99%)的文章都在探討第一點，如何讓 `Kubernetes` 內部的容易有網路服務，不論是容器間，容器存取外部或是容器被外部存取。

而第二點其實仔細分析後，其概念最後也是回歸到第一點的需求，只是網路服務提供商內部的容器對於網路效率的要求更嚴苛，譬如更高的流量，更低的延遲，更多的網路介面等。

這些點我們之後再來仔細討論

## Before Kubernetes
如同先前提到的儲存資源的概念一樣，網路概念本身也是包山包海，各式各樣的議題可以討論，包含了層級也是非常的廣，譬如
1. 硬體交換機的設計與建制
2. 網路架構的佈建，包含了各式各樣的拓墣 (Fat-Tree, Leaf-Spine..etc), 交換機內的連線 (LAG, MC-LAG, Bonding)
3. 各式各樣的路由技術或是路由議題 (BGP, OSPF, DSR, RIP, ECMP..etc)
4. 各式各樣的網路協定 (IPv4/IPv6,Unicast/Multicast/Broadcast,TCP,UDP,ICMP,MPTCP,QUIC)
5. 以`Linux`為範例來說，軟體上也有各式各樣的網路封包處理，譬如常用的 `iptables/tc`, `linux bridge`, `tun/tap`
6. SDN 概念的管理與佈建 (SDN Controller, SW/HW Switch, P4, ONOS..etc)
7. 各式各樣的邏輯網路部署 (VLAN, VXLAN, GRE, NVGRE)
8. 效能優先的網路技術，如 DPDK, RDMA, Smart NIC 等
9. ... 基本上講不完，包含的議題實在太多了

上述每個領域都有各自的廠商/軟體在從事這方面的研究，這些領域要互相整合來提供一個更為強大的網路架構才是真正有價值的部分。

所以只要仔細想一下， `Kuberentes` 本身本來就不可能一口氣支援上述的所有的功能，甚至每個都處理的完美無缺點。 這對於整個 `Kubernetes` 平台來說是一個多麽大的負擔，可以說是一個不切實際的理想。

{% note danger %}
就跟儲存一樣，請放下 `Kubernetes` 是萬能的想法，不是套上 `Kuberentes` 什麼網路問題都解決了，請不要給 `Kubernetes` 過多錯誤的期待與期盼
{% endnote %}


## Kubernetes
對於 `Kubernetes` 來說，我個人的認知下， `Kubernetes` 在網路的部分比`儲存` 的地方做了更多的支援，除了標準介面之外，也有部分是 `Kuberentes` 自行實現的功能。 

如同 `CSI(Container Storage Interface)` 這個針對 `儲存` 所定義的標準介面，在網路部分也有與之對應的 `CNI(Container Network Interface)`.  `kubernetes` 透過 `CNI` 這個介面來與後方的 `網路解決方案` 溝通，而該解決方案(我底下就統稱 `CNI`比較方便) 就我自己的開發經驗與體悟，我認為 `CNI` 最基本的要求就是在`在對應的階段為對應的容器提供網路能力`，就這樣非常簡單。

但是什麼叫做 `提供網路能力`, 這個部分我認為沒有定義，畢竟誰說網路一定是走 `IPv4` ? 誰說網路一定要至少到 `Layer3 IP` 難道不能 `point ot point` 互連嗎?
主要是因為這部分的功能特性對於大部分的使用者都沒有需求，而目前最常見也是 `IPv4 + TCP/UDP` 的傳輸方式，因此才會看到大部分的 `CNI` 都在講這些。

這邊使用實際需求來探討一下 `CNI` 所做的事情，假設我們希望所有容器彼此之間可以透過 `IPv4` 來互相存取彼此，不論是同節點或是跨節點的容器們都要可以滿足這個需求。
在這個要求下，最常見的步驟如下
1. 容器創建之時，想辦法獲得一個 `IPv4` 位置，並且將該 `IPv4` 位置分配到容器內
2. 幫容器與外部節點中間建立一個能夠聯繫的通道
3. 設定相關的路由條件 (overlay? underlay?)

上面三個步驟，實際上做法百百種
1. 如何取得 `IPv4`？ 如果要取得不重複的 `IP` 該怎麼做，需要有集中式的管理？ 還是分散式各自管理? 
2. 如何讓`容器`與`外部節點`有聯繫的通道? 要走 `veth`？ `host-device` ? `直接掛載網卡進去` ?
3. 如何設定路由條件? 動態路由協議還是靜態路由協議? 要透過集中式資訊傳遞 `gateway` 嗎? 

{% note danger %}
容器間到底怎麼傳輸的，需不需要封裝，透過什麼網卡，要不要透過 NAT 處理? 這一切都是 `CNI` 介面背後的實現，對於 `kubernetes` 來說其實根本沒有想要，也沒有能力去處理這些。
所以不要再幻想 `kubernetes` 能夠為你建立各式各樣的網路環境了
{% endnote %}


除了上述的容器間封包傳遞外，還有其他的網路議題
1. 外部網路存取容器服務 (Service/Ingress)
2. DNS 服務
3. ACL (Network Policy)

這三個部份中， `CNI` 都多多少少有涉獵其中，譬如前兩點就會依賴 `Kubernetes` 提供的網路抽象層 (Service/Ingress)來使用，而這些部分的最底層則是 `CNI` 要提供基本的容器存取能力來打造良好的基底，上述的網路抽象層才能夠正常運作。

{% note danger %}
kubernetes 在 Service/Ingress 中間自行實現了一個模組，大抵上稱為 `kube-proxy`, 其底層可以使用 `iptables`, `IPVS`, `user-space software` 等不同的實現方法，這部分是跟 `CNI` 完全無關。
所以不要再看到 `IPVS` 就覺得好像是什麼全新的功能，其實最原生的概念就只是中間抽象層功能的一種實現而已。
{% endnote %}

而 `ACL` 則是一個完全抽象層， `Kubernetes` 本身只實現接口，不實現底層功能，因為 `kubernetes` 沒有任何頭緒你的 `CNI` 是如何讓容器有網路能力的，因此 `kubernetes` 根本沒有辦法幫你去設定相關的 `ACL`，則要依賴 `CNI` 自己去完成了。

## Summary
1. Kubernetes 本身有定義 `CNI` 這個網路標準介面，同時也有定義網路服務的中介層
2. `CNI` 面對的網路提供方案自行想辦法實作功能，讓容器有網路連線能力
3. Kubernetes 本身也有定義的中介層 `Service/Ingress` 並且透過不同的模組來提供此功能 `iptables/IPVS`.
4. `CNI` 跟 `Service/Ingress` 是會衝突的，也有可能彼此沒有配合，這中間沒有絕對的穩定整合。
5. 遇到網路任何問題，可能是 Kubernetes 整合上的問題，也有可能是 `CNI` 本身的問題，這部分要仰賴管理者的經驗來處理，不可能也沒辦法一定概括誰的問題。


若對於 `CNI` 標準有興趣的可以參閱下列文章
- [Containernetworking CNI github](https://github.com/containernetworking/cni
)
- [CNI 系列文章](https://www.hwchiu.com/tags/CNI/)

對於 `Kubernetes` 抽象層可以參閱下列文章了解其原理
- [What is Service](https://www.hwchiu.com/kubernetes-service-i.html)
- [How to Implement Kubernetes Service - ClusterIP
](https://www.hwchiu.com/kubernetes-service-ii.html)
- [How To Implement Kubernetes Service - NodePort](https://www.hwchiu.com/kubernetes-service-iii.html)
- [How To Implement Kubernetes Service - SessionAffinity](https://www.hwchiu.com/kubernetes-service-iiii.html)
- [Introduction to Kubernetes Ingress](https://www.hwchiu.com/ingress-1.html)


# Computing(運算)
最後則要講到`運算能力`這個部分了，眾所皆知的就是 `Kubernetes` 是一個 `Container(容器)` 管理協作平台，因此上面的基本運算單位都是基於 `Container(容器)` 來管理.

不意外的是， `Kubernetes` 本身又搞了一些相關標準來處理這個部分，如 `CRI (Container Runtime Interface)` 或是 `Device Plugin` 等相關的標準。

`Container(容器)` 與 `Virtual Machine(虛擬機器)` 之間的討論與比較一直以來都沒有停過，推薦這篇文章 [Container vs VM: When and Why?
](https://igene.tw/container-vs-vm)， 以光譜兩端來說，除了完全的`Container`以及完全的`Virtual Machine`之外，也有愈來愈多的混合體，希望可以結合 `Container` 以及 `Virtual Machine` 各自的優點得到一個更好的虛擬化環境。 譬如(這邊不討論細節)
1. gVisor
2. Kata Container

對於 `kubernetes` 來說，其實本身並不在意到底底下的容器化技術實際上是怎麼實現的，你要用 `Docker`, `rkt`, `CRI-O` 都無所謂，甚至背後是一個偽裝成 `Container` 的 `Virtaul Machine` [virtlet](https://github.com/Mirantis/virtlet) 都可以。

## Container
{% note danger %}
就我個人的觀察來說，最多人在這個議題最大的誤解就是 `容器` 是萬能的 
{% endnote %}

很多人看到的容器化可帶來的優勢後，一股腦地就要所有東西都容器化，完全沒有去思考到底`為什麼自己本身的服務需要容器化，容器化可以帶來什麼優點`. 
舉個例來說，我想要透過 `SRIOV` 等相關硬體設定分配給我的應用程式使用時候， `Virtual Machine` 方面的發展與支援就遠比 `Container` 來的好多. 當然 `RDMA` 我個人也是抱持者 `Virtual Machine` 支援更好的情況來看待。

很多人踩中了第一點認為 `容器` 是萬能之後，就會開始進行`要命的第二步驟`，就是將原先的應用程式`容器化`. 太多太多的人都認為只要寫一個 `Dockerfile` 將原先的應用程式們全部包裝起來放在一起就是一個很好的`容器` 來使用了。

這就是常常會看到有一些的 `Dockerfile` 內同時跑了一堆`Daemon(守護行程)` 的容器，然後彼此之間相互依賴，對於外部的 `Signal` 以及 `生命週期確認目標` 都沒有辦法搞得清楚

最後在使用的時候又會發現各種軟體版本相依，日誌混雜，升級麻煩，監控重啟等都遇到各種問題，然後又會產生各種 `Workaround` 來使用。

最後就會發現根本把 `Container` 當作 `Virtual Machine` 來使用，然後再補一句 `Contaienr 根本不好用啊`.


## GPU
除了 `Container` 本身的使用之外，還有一個近年因應 `AI` 產業蓬勃發展而對於 `Kubernetes` 最大的誤解。
就是 `GPU 虛擬化`

由於 `AI` 的蓬勃發展，帶動了 `GPU` 的使用需求, 而眾多的運算框架本身在使用 `GPU` 都是以 `GPU張數` 作為基本單位來使用的，譬如一台機器上面只有 `一張 GPU 卡`，同時間就只能有一個應用程式來使用該 `GPU` 並且使用一張 `GPU` 卡。

當然 `Nvidia` 本身也有推出一些虛擬化解決方案，譬如 [Virtual GPU](https://www.nvidia.com/en-us/design-visualization/technologies/virtual-gpu/) 來解決這方面的問題，希望能夠讓最上層的虛擬化環境可以不需要考慮底層的 `GPU` 真實數量。
除此之外也有一些廠商，如 [Bitfusion](https://bitfusion.io/) 等也有提供對應的 `GPU` 虛擬化解決方案，底層還會使用如 `GPUDirect RDMA(supported by Nvidia)` 等技術來提供快速的 `GPU Pool` 概念。

但是仔細思考上面的文件，會觀察到這類型的技術都還是廠商自行研發或是提供介面來開發，而這部分牽扯到 `kubernetes` 之後又變得很有趣了。

`Kubernetes` 本身是依賴 `Device Plugin` 這個`外部裝置`的標準介面來存取`外部裝置`, 包含了 [GPU](https://github.com/NVIDIA/k8s-device-plugin), RDMA, SRIOV 等

`NVIDIA` 針對 `Device Plugin` 開發的一個簡單的 `GPU` 分配模組，可以將節點上的 `GPU` 分配到 `Container` 內部，而目前就是以`張`為基本單位去分配，因此 `Container` 可以看到的 `GPU` 就是真實主機上面的 `真實數量`。

然而由於 `AI` 的需求發展，加上 `kubernetes` 聽起來很棒的說法，兩者結合再一起之後就產生了無懈可擊的期盼，`kubernetes 能夠虛擬化 GPU卡，讓容器同時存取多台節點上的 GPU`

舉例來說，節點上只有 `2張` GPU 卡，卻總是期盼一個要求`四張 GPU`卡的容器可以正常運作並且 `Kubernetes` 有一個漂亮的演算法可以從叢集中自動分配 `GPU卡` 來盡可能的提升使用效率.

光問 `GPU` 之間彼此怎麼溝通，這個事情就不是一個好處理的事情，透過 `Process` 不停地交換資料導致大量的 `IO Copy` 反而會造成效能下降， 若要透過 `GPU` 直接跨節點交換資料，又要透過 `GPUDirect RDMA` 等技術來處理，而這個部分又是要 `GPU 應用程式` 自行處理。

除此之外還有很多相關的議題要處理， `kubernetes` 本身根本不是因應 `GPU` 使用需求而產生的平台，抱有 `kubernetes` 可以完全處理一切問題實在是擁有太過美好的幻想了。



## Summary
- 運算容器方面， `Container` 與 `Virtaul Machine` 都有各自的使用情境，不要一昧追求 `Container` 或是 `Virtual Machine`， 請多多思考自己的需求
- `容器化` 不是把直接 `Virtual Machine` 的使用習慣換個環境使用就叫做 `容器化`，而是要從概念上去暸解與使用
- 千萬不要為了容器化而容器化，不是用來解決問題的改動都只是在製造更多的問題。
- 硬體裝置部分，`GPU` 很夯沒錯，但是 `GPU` 本身的複雜程度沒有這麼好處理， `kubernetes` 不是為了 `GPU` 而誕生的，沒有辦法什麼都辦得到，要特定的功能還是想辦法自己去修改 `kubernetes` 或是搭建其他的服務來處理


# Summary
講了這麼多，想傳達的就是
{% note danger %}
Kubernetes 不是萬能的，作為一個平台其強大之處在於透過各式各樣的介面與框架來相容第三方解決方案的整合。
在此模式下彼此都可以專注於自身能力的發展並且互相合作來提供更好的使用價值

不要對 Kubernetes 抱有太大的期盼，不要一昧的跟風，看到什麼是主流就馬上換什麼，請好好的思考到底自己需要什麼，到底 `kubernetes` 帶來的價值是否值得採用

儲存，網路，運算 這些常見的使用資源中，請仔細研究與考慮自己使用情境的需求，不要一昧看到大家說什麼就用什麼，最後苦的只是自己。
{% endnote %}


# Reference
- https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md
- https://github.com/containernetworking/cni
- https://www.nvidia.com/en-us/design-visualization/technologies/virtual-gpu/
- https://github.com/NVIDIA/k8s-device-plugin
- https://bitfusion.io/
