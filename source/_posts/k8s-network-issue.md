---
title: 從網路觀點來看導入 Kubernetes 的可能痛點
keywords: 'Kubernetes,Network,CNI,缺點,問題'
tags:
  - CNI
  - Kubernetes
  - Network
description: 探討 Kubernetes 架構下網路能力的不足，導入 Kubernetes 時有可能會無法解決的網路架構問題。
abbrlink: 62977
date: 2021-04-25 11:16:06
---

# 前言

隨者 Kubernetes 容器管理平台的風行，愈來愈多的行業開始踏入到容器化的世界。許多第一次接觸容器化的使用者都會看到各種關於容器化的好處，除了跟 Bare Metal 直接相比外，VM 這種重量級虛擬化的技術也是非常容易被拿來比較的對象。

談論到 VM 的使用方式時，最常使用的方式就是透過公有雲的環境來架設環境，任何規格的機器都可以於彈指之間創建完成，但是有一部分的產業與使用者卻沒有辦法這麼輕易的將所有需求轉移到公有雲上，可能會必須要於地端機房中透過 OpenStack 等方式來自行管理 VM。

導入 Kubernetes 時有一個很重要的事情就是，團隊到底有沒有辦法一次到位？通常情況下是不太可能一步到位。這意味者整個導入過程中會有一個過渡時期，架構中同時會同時存在舊有架構以及  Kubernetes，要如何確保舊有的商業邏輯不被影響則是導入 Kubernetes 中極為困難的問題。

本篇文章則會從網路架構的角度去探討不同的使用議題，而這些議題目前有哪些可能的解決方案。

註: 這些困境並不一定每個團隊都有需求，只是單純從技術層面探討到底 K8S 的網路架構還有哪些不足的部分。

# 困境

## 固定 IP/MAC

過往透過 Bare Metal 或是 VM 來管理服務時，固定 IP/MAC 是個常見的做法與需求。
使用 DHCP 管理 IP 分配與發送的架構會希望 VM 可以綁定 MAC 地址，如此一來才可以確保該服務可能使用相同的 IP 地址(即使 VM 遷移到不同實體機器)。
而採用靜態 IP 部署的則會希望服務 IP 固定。

基於上述的設計，公司團隊通常習慣所有服務都會有個固定 IP，因此內部防火牆或是相關服務都會透過使用這些 IP 來處理流量。

當這些服務準備透過容器來處理時，會發現就算是透過 docker container 都很難處理，何況是 Kubernetes 更為複雜的管理平台。
許許多多的文章都會告訴你，於容器化的世界中不要再執著於固定 IP 了，請改用 DNS 的方式來存取服務，譬如 Kubernetes 內的 Service 服務。

使用這類型的解決方案從邏輯上看可以滿足大部分需求，實務上則沒有想像的這麼強大，以上述的內部防火牆為範例，可能的失敗原因有

1. 這類型的 DNS 紀錄預設都是由叢集內的 CoreDNS 來管理，這意味外部服務譬如公司防火牆根本沒有辦法取得這些資訊。
2. 將 DNS 服務串連可以正確存取後，有可能防火牆不支援使用 DNS 設定，只能使用 IP 地址。
3. Kubernetes Service 回傳的 IP 地址是基於 ClusterIP 的設計，其指向的是一個 virtual IP，本身是沒有辦法被外部訪問的，因此本來就不會有任何外部流量使用 ClusterIP 來存取 Kubernetes 內的服務，這種設定本質上反而沒有意義。
4. 若透過 NodePort/Load-Balancer 等方式來設定網路存取，則防火牆面對的對象就不是原生應用程式 IP，而是中間層的 LoadBalacner 或是所有節點(NodePort)。如果 Port Number 會隨者部署更新而改變時，防火牆內的設定就過期了。
5. 另外一種可能性就是直接打通所有網路，讓 Contaienr Pod 的 IP 直接與現有架構是相同網段，這樣外部服務可以直接透過 PodIP 存取該應用程式，但是問題就會回歸到原本狀態，每次 Pod 重啟產生不同 IP 使得防火牆沒有辦法設定。


除了上述範例外，如果應用程式本身使用的協定不是常見的 TCP/UDP 這種，譬如 SCTP。只要 Kubernetes 內的 Service 沒有提供成熟的支援，有可能發生 SNAT/DNAT 沒辦法正常運作，這種情況下會希望能夠讓該服務有一個可直接存取的固定 IP，不需要透過 ClusterIP/NodePort 等服務存取。


## 主從網路架構

第二個要探討的則是主從網路架構，甚至可以說多重網路架構。
舉一個範例，架設 Ceph 這套分散式儲存架構時，會針對流量設計兩種不同的架構，分別處理 Control Plane 以及 Data Plane。

Control 可以想成 Ceph 架構下所有伺服器上元件互相溝通用的網路，而 Data plane 則是專門用來傳送真正使用者需要的資料。

透過這種分層架構的設計，可以讓 Control/Data 兩種流量分開處理
1. 不會因為 Data 流量過大導致整個網路頻寬被塞爆使得 Control 流量沒有辦法正確交換。
2. 同時針對兩種不同需求，網路架構也可以分開設計，譬如 Control Plane 採用 1G 網路而 Data Plane 則是使用 100G 網路。

如果團隊內本來的應用環境也有這種需求的話，下一個問題就是，導入 Kubernetes 後該怎麼處理?

熟悉 Linux 網路的人可能會想說這個問題可以透過 Routing Table 的概念來處理，對 k8s 的節點設定不同的 Routing table 規則，讓封包可以根據不同的 destination IP 網段走不同的網卡。但是這個方法其實有一些令人困擾的事情
1. 目前 Kubernetes 並沒有這種機制，要達成就則是管理人員要想辦法去完成這些設定。設定的同時也要考量到新節點加入後也要自動被設定，同時每個節點上的網卡名稱也有可能不同。
2. 不同 CNI 如何實作跨節點 Pod 存取的方式不同，譬如使用單純 Routing，透過 Tunnel 技術(VXLAN...etc)等不同技術。也因為這些網路底層的技術不同，因此如何設定這些 Routing rules 使用的策略也都完全不同


## 多租戶網路需求

第三個要探討的需求是多租戶的隔離政策，熟悉 Kubernetes 的朋友都知道 Kubernetes 透過 namespace 的概念提供一個隔離的用法，然而 namespace 實際上完全只是一個邏輯上的隔離，所有的使用資源(運算網路儲存)彼此之間並非真正隔離。

假設今天想要透過 Kubernetes 打造一個服務多位使用者的平台，使用者透過 Container 來運行所需的工作，這種情況下要如何達到網路的隔離?

Kubernetes 設計了 NetworkPolicy 的標準格式，仰賴各種 CNI 的實作來提供 Kubernetes 元件之間的防火牆，然而我認為透過 NetworkPolicy 並沒有辦法達成真正的網路隔離，原因有

1. 採用的 CNI 如果沒有實作這方面的功能，NetworkPolicy 就不能運作
2. 基於 IP/Port 的防火牆只能說是防火牆，沒有辦法當作真正的隔離，相同網域下的其他流量封包如廣播風包是否還是有機會竄流到其他的 Pod 中？ 透過這種方式是否有辦法癱瘓其他 Pod 的網路功能，造成其沒有辦法收送真正的流量
3. 相對於過往使用 VLAN TAG 等方式來隔離用戶間的流量，甚至透過 Trunk 來設定更為複雜的隔離關係，想要透過 NetworkPolicy 來實現基本上是非常痛苦且困難的



# Kubenetes 網路功能

對於三個困境有些許頭緒後，接下來要思考的就是，如何於 Kubernetes 的環境中解決三個困境。

要解決這個問題，必須要先理解到底 Kubernetes 內的網路功能是如何實作的，透過對 Kubernetes 實作的理解，才有辦法仔細的思考可能的解決方案。

就我個人對 Kubernetes 的理解，目前網路功能由兩大塊元件組成，然而這兩個元件的維護是分開的，因為分開也造就整合上通常會有落差，目前能夠同時兼顧這兩塊的解決方案並沒有太多。

1. Kube-Proxy
2. CNI

## Kube-Proxy
Kube-Proxy 最簡單的功用就是 Service 的實作，如何把 ClusterIP,NodePort 這些 IP:Port 給轉換到最終的 PodIP/Port 則是 Kube-Proxy 要完成的事情。
目前內建的實作主流有兩派，分別是基於 iptables 與 ipvs 兩種不同的方式。
這兩種方式都是透過 Kernel 內的 conntrack 配上各種 netfilter 的模組來完成 service 的轉換

註: iptables 與 ipvs 主流上最大的差異是效能，有興趣瞭解差異的可以參考我之前線上演講影片，有詳細介紹兩者的底層架構與實作方式

這邊要注意的， kube-proxy 幫忙設定好的這些規則，目的是**幫忙轉換 ClusterIP, NodePort 到 Pod IP(這邊會發生一次的負載平衡的選擇)**，至於封包到底如何於多節點之間轉送則不是 kube-proxy 要處理的，而是讓 CNI 幫忙處理。

## CNI
CNI(Container Network Interface) 基本上需要負責兩件事情，分別是
1. IPAM (IP Address Management)
2. Network Connectivity (如何讓 Pods 可以互相溝通)

上述兩個概念真正要做的事情非常簡單，就是 **如何讓你節點上的 Pod 可以存取到另外一個節點上的 Pod**，所以第一件事情就是幫你的 Pod 分配一個 IP 地址，接者想辦法讓不同節點之間的網路可以打通。

以下顯示了兩種可能的架構模型

第一種模型的情境是，你想要將不同地區的節點創建出一個大 Kubernetes 叢集，如下圖範例。
叢集中有三個節點，分別位於 Taipei,Hsinchu 以及 Taichung。每個節點自己本身所屬的機房都有自己的網段，分別是 192.168.56.0/24, 10.23.45.0/24 以及 10.54.78.0/24，而這三個節點本身的 IP 地址也屬於這三個網段(是節點IP，跟 Pod 沒有關係)
![](https://i.imgur.com/8sXLjdz.png)

這三個節點的機房本身要先處理好網路的問題，確保這些節點彼此之間可以互通，這些互通的過程我統稱為WAN 網路(Underlay Network)。

三個節點上都要安裝 CNI 的應用程式(k8s CNI 是節點為單位，你可以發現所有CNI都使用 Daemonset 來安裝相關檔案)，這些 CNI 要想辦法幫各自節點上的 Pod 去維護 IP，並且基於已經打通的 underlay network 再去維護一層 overlay network。


而第二個模型則是，假設所有的 k8s 節點物理位置屬於同一個節點，譬如 Hsinchu，因此三個節點的網段都屬於 10.23.45.0/24

這種情況下，三個節點彼此之間的溝通會簡單很多，由於都屬於同一個機房網段中，我使用 LAN(Underlay) 來稱呼
![](https://i.imgur.com/P126KoV.png)

CNI 的工作跟上述架構，不論底層架構長怎樣，都需要幫忙分配 PodIP 地址，並且打通彼此。

就實務上的經驗，大部分的情況都是基於第二種架構，大部分跨地區的叢集都會是獨立一套自己的 k8s 叢集，而不會讓彼此節點跨地區。


針對上述的架構，可以衍伸出至少三種不同的 IPAM 處理方式，分別如下

第一種是最常見的作法，也是目前自架叢集時 CNI 會採取的策略，針對整個 Cluster 給予一個很大的虛擬IP地址，譬如範例中的 10.123.16.0/20，意義是整個 k8s 叢集內的 Pod 都必須要符合這個網段

接者為了方便與管理，同時基於每個 Pod 的 IP 要唯一不衝突，每個 k8s 節點會需要設定能夠容量的 Pod 數量上限。針對這個上限反推出需要使用的網段大小，譬如大愛 256 個Pod 就需要 /24 這樣的網段，而這個網段必須符合叢集範圍 10.123.16.0/20。

由於叢集的範圍是/20,每個節點需要/24，因此差距就是(24-20)，換算出來就是 2^4=16。
這意味整個 k8s 叢集只能有 16 個節點，每個節點 256 個 Pod，整個叢集容量就是 4096 個 Pod。

![](https://i.imgur.com/DzXngJa.png)

範例中可以看到三個節點分別分配了 10.123.24.4, 10.123.25.4 以及 10.123.26.4 的IP，精準地講可以說這三個節點上的Pod IP 範圍必須屬於 10.123.24.0/24, 10.123.25.0/24 以及 10.123.26.0/24。

由於 10.123.16.0/20 是一個私有的IP，每個使用者的叢集都可以有一個屬於自己的 10.123.16.0/24 網段，因此如何讓節點A上面的 10.123.24.4 跨縣市存取到節點C上面的 10.123.26.4，這部分就是 Overlay 出馬的時候了。

CNI 根據設定與需求，幫忙搭建出 Overlay 網路，最簡單的概念就是透過封裝(Encapsulating)協定，將Pod的網路封包給封裝起來，常見的 Flannel 就會使用 VXLAN 來進行封裝，其他解法也有 IPIP(IP in IP Encapsulation) 等不同協定。

註:
1. 這種 IP 分配方法直覺且簡單，唯一要注意的是 Pod & Node 的數量
2. Pod 如果重啟被分配到不同節點，被分配到的 IP 網段直接不同，因此 IP 一定不同。


第二種架構很類似，也是先定義一個基於 k8s 叢集的網段，所有的 Pod 都要符合這個網段。接者希望這個網段是供所有節點共用，節點之間不再去區分彼此，簡單來說。
1. 不去考慮節點最多能分配多少 Pod
2. 所有節點直接共用 /20 個網段

下圖範例中，節點A上的Pod使用 10.123.24.4 而節點 B 上的Pod則使用 10.123.24.7，兩者都屬於 10.123.16.0/20 的網段。
![](https://i.imgur.com/38Me1Ye.png)

這種架構帶來的好處就是 Pod 不論分配到哪個節點，都可以擁有相同的網段，意味者有可能保留相同的 IP。

但是此架構帶來的壞處就是，非常難實作與管理。
1. 每個節點上的 CNI 都是獨立運作，要如何確保每個 CNI 獨立運作時不會發生 IP 分配衝突的問題。這意味必須要有一個集中管理的資料庫來分配與計畫
2. Overlay Network 的管理必須要以 IP 為單位去設計，不能以網段為單位去設計。

我之前有嘗試使用 k8s 內建的 etcd 作為資料庫來完成這種範例，但是使用情境其實不太多，單純是一種網路架構的探討與研究，同時底層 CNI 也必須要自己重新實作來支援這種分配。

最後一種架構則是雲端業者最喜歡提供的類型，直接將 Pod 能夠使用的網段與節點的網段直接串一起。

就**網路是否能通，是否可以實作**的概念來下，前述兩個概念都可以與第三種整合，也就是
1. Pod 與節點共享網段，但是每個節點會在分配自己的一個小網段。
2. Pod 與節點共享網段，但是每個節點彼此不會有自己的小網段，大家直接攤平使用。

假設今天透過 AWS 的服務來架設 VM，並且於 VM 上面部署 Kubernetes 叢集。能夠讓 VM 與 K8S Pod 使用的網段全部都是基於 VPC 的設定，這種架構非常仰賴底層網路運作以及 CNI 的設定

此架構帶來的好壞處
1. 網段共享，管理人員少一層網段需要管理，直接透過一個大網段管理節點與 Pod
2. 任何能夠針對該網段處理的服務現在都可以針對 Pod 使用
3. 封包經過愈少 NAT的轉換，除錯方面會更簡單(但是因為你使用的是別人的底層架構，其實除錯還是很麻煩，因為自己能夠碰到的地方太少)。
4. CNI 本身都要客製化，針對不同雲端業者去設計，因此 GCP/Azure/AWS 都有設計自己的 CNI。
5. 至於這些 CNI 想要如何分配 IP，要攤平共享還是依賴節點去分配，則是依賴每家業者的實作。
![](https://i.imgur.com/alJLUHX.png)

# 困境分析

看完上述分析，對於 Kubernetes 的網路有基本概念後，接下來就要回到主題的三大困境，來聊聊這三大困境為什麼這麼難滿足

## IP/MAC 固定

1. IP的分配是由 CNI 提供的，因此 CNI 本身若不提供靜態IP的分配 -> 基本上沒救
2. 如果 IP 是基於節點為單位，則固定IP 的Pod不能夠換節點，否則整個網路都不會通
3. 目前 CNI 官方維護的三種 [IPAM](https://github.com/containernetworking/plugins/tree/master/plugins/ipam)，分別是 DHCP, Host-local 以及 Static
    - DHCP -> 就是 DHCP，但是使用上要先考慮你的 Pod 是否有辦法收到 DHCP 封包
    - Host-local -> 以節點為單位的分配方式， Flannel 實際上是偷偷呼叫 host-local 來完成 IP 分配
    - Static -> 固定 IP，標榜就是測試用。實務上很難整合，原因是當 Pod 要創立時， kubelet 會叫起 CNI 來幫忙分配，而 CNI 要怎麼知道哪些 Pod 要用 Static，哪些要用其他的 IPAM?
4. 沒有客製化 CNI 的話，很難滿足此需求

此外固定 IP 的架構還要考慮到
1. 基本上不可能使用多重副本的概念，否則 IP 必定衝突


## 主從網路架構
主從網路架構下， Underlay 網路可能會有多條，架構可能變成如下圖
![](https://i.imgur.com/o2YhjEu.png)

整個底層網路分成 Control Plane 以及 Data Plane，網段分別是 10.23.45.0/24 以及 10.23.44.0/24。

這時候第一個問題就是，什麼流量走 Control Plane，什麼流量走 Data Plane，範例如
1. Pod 之間的溝通全部走 Data Plane
2. 除此之外的流量全部都走 Control Plane，譬如 API-Server, Scheullder, Controller 之間的溝通

如同前述提到的， Pod 與 Pod 之間該怎麼溝通是 CNI 負責搞定的，如果 CNI 本身根本不知道底層有兩條網路可以選，根本沒有辦法處理。

大部分的基本上 CNI 都是仰賴 Linux Kernel 內的 Routing Table 來處理 Underlay 的轉發，因此有些人會思考到利用修改 Routing Table 的方式，讓 Kernel 知道什麼樣的封包要走哪張網卡出去，也就是走哪種網路出去。

上述的思路要考慮的點
1. 如果 CNI 採用的是封裝技術的話，則封裝過程必須要知道目標節點要使用的是屬於 Data Plane 的網段，不能使用 Control Plane 的網段
2. 如果 CNI 採用的 Routing 來處理的話，則也必須要知道 Data Plane 使用的網段與節點資訊

簡單來說，如果 CNI 本身安裝時可以設定 Underlay Network 的資訊的話，這個部分不會是太大的問題，而滿多 CNI 的確也都支援這方面的設定。

但是如果今天上述的需求改成 "特定的Pod走 Data Plane，特定的 Pod 走 Control Plane" 或是 "特定的 Protocol:IP:Port 走 Data Plane, 剩下走 Control Plane" 這種很彈性的設計，則 CNI 也很難處理。
就如同前述所講，CNI 就是創立 Pod 時就會被呼叫起來去設定IP與網路，因此架構是以 Pod 為單位，沒有辦法針對應用程式或是特定的 Pod 進行細部顆粒的處理。


## 多從租戶需求

從上述的架構來看， CNI 就是基於 Underlay 的架構下，搭建出一個能夠讓不同節點Pod互通的 overlay 網路。 CNI 本身被呼叫時也沒有太多關於各別 Pod 的資訊，唯一有的只有 Pod Sandbox ID 這種非常粗略的資訊，因此想要 CNI 可以理解 Pod 與使用者的關係基本上只能重寫 CNI，引入這個機制與概念才有可能解決，否則預設的 CNI 根本不理解這種概念。

CNI 有能力知道每個 Pod 分配的IP(因為就是CNI分配的)，同時也有能力去搭建出串連彼此的 Overlay 網路，因此 NetworkPolicy 的引入讓 CNI 能夠針對網路流量的防火牆貢獻一點微薄之力。

Network Policy 部分如何實作完全取決於 "CNI 如何實作 Network Connectivity"，如果是完全仰賴 Linux Kernel 來處理封包流向的，就有可能會透過 iptables 的規則來阻擋封包傳遞。

所以對於使用者來說，假如透過 Network Policy 去設定某兩個 Pod 彼此之間不能互通，這些 Pod 的 IP 會被轉換成對應的 iptables 規則並且設定。一但 Pod 被重新部署導致 IP 改變，則 CNI 也要有能力偵測到變化並重新設新的規則

註:精準的說，CNI 是一個單次呼叫的 Process，而 CNI 解決方案通常會包含 CNI 執行檔案以及對應的 Controller，這個範例中指的是對應的 Controller。

總結來說，從網路的角度來看， CNI 是完全有辦法去做到很強大的網路隔離，但是 CNI 的架構讓其很難理解 Pod 與使用者的關係，所以想要達成 "基於使用者的網路隔離" 是非常困難的。
不過軟體很難說有什麼辦不到的事情，自己撰寫 CNI 就有辦法達成，只是成本效益的取捨而已。

# 可能解法

上述三個問題各別處理都很麻煩，何況想要同時解決三個問題，根據我過往的經驗與理解，目前架構下要完成這個議題最有效的方式就是下列兩種方式
1. 混搭使用 CNI
2. 抽離 Kubernetes

思考的起點很簡單，根據
1. CNI 實作百百種，每種實作都有自己適合的場景，如果可以根據不同的需求(譬如不同的Pod)而呼叫不同的 CNI，這樣就可以彈性的去滿足 IPAM 以及網路傳輸。
2. 同時如果 Kubernetes 目前就是沒有辦法順利處理這種網路問題，那就想辦法將網路的管理給往外延伸。讓 Kubernetes 專心處理 Pod/Container 的處理，而這些特別的網路需求則讓擅長的解決方案來處理。

## 混搭 CNI
重新複習一次，根據目前 k8s 的架構，kubelet 會於創建/刪除 Pod 的時候去呼叫 CNI 來處理網路需求。由於沒有辦法很動態的去決定每個 Pod 要呼叫哪種 CNI，那就撰寫一個全新的 CNI 來提供這種功能，這種 CNI 俗稱為 metaplugin CNI。

Metaplugin CNI 可以根據需求針對單一 Pod 呼叫多種不同的 CNI 來設定網路，所以整個架構變化如下。

下圖中忽略底層的程式呼叫細節(CRI)，主要是專注於 CNI 的呼叫邏輯。
上方呈現的是本文一直探討的 CNI 架構，也是幾乎所有 K8s 玩家會使用的架構，透過一套 CNI 來處理 k8s 叢集內的所有 Pod。
![](https://i.imgur.com/NMklAAn.png)
下方呈現的則是 Metaplugin CNI 架構， Metaplugin CNI 本身有額外的設定檔案，該檔案中描述支援哪些 CNI，以及彼此的呼叫順序。當 Pod 被創建時， Kubelet 就會把 Pod 的資訊傳給 Metaplugin CNI，該 CNI 則根據設定檔依序呼叫多個 CNI 來處理相同的一個 Pod。

舉例來說，實務上的使用流程為
1. 安裝 Metaplugin CNI
2. 設定想要使用到的所有 CNI
3. 創建 Pod 時於 Annotation 內去描述希望該 Pod 要使用哪些 CNI。
4. Metaplugin 被呼叫時，會去抓取到該 Pod 描述的資訊，接者呼叫要使用的 CNI


常見的 Metaplugin CNI 解決方案有 [Multus-CNI](https://github.com/k8snetworkplumbingwg/multus-cni) 以及 [DANM](https://github.com/nokia/danm) 兩種解決方案。

透過這種架構， K8s 可以獲得更彈性的 CNI 架構，譬如可以讓單一 Pod 裏面創建多個網路介面，同時有不同的 IP 設定。

以下是兩個使用範例(只是範例，不代表真的會這樣用)

1. 系統內安裝兩種不同的 CNI，這兩個 CNI 設定使用不同的 overlay network，創建 Pod 的時候根據需求決定要使用哪套 CNI 就可以達成 "根據 Pod 來決定要讓其流量走哪條網路"。

這種架構下每個 Pod 內都只會有一張網卡，所有從該 Pod 出去的流量就會走特定的網路出去。
![](https://i.imgur.com/nVEgGmF.png)


PodA 與 PodC 屬於同一個網路，而 PodB 與 PodD 屬於同一個網路，但是 PodA 不一定可以跟 PodB 溝通，這部分主要取決於 Underlay 網路的設計。


2. 系統內安裝兩種不同的 CNI，這兩個 CNI 設定使用不同的 overlay network，創建 Pod 的時候同時呼叫兩種 CNI，接者 Pod 裡面透過 Routing Table 來決定封包該如何轉發。

第二種架構比較常見，讓所有的 Pod 同時都擁有兩種網路能力，Pod 內的應用程式根據 Routing Table 的規則來決定封包該如何轉送，範例如下兩圖。
![](https://i.imgur.com/at8aYOy.png)


![](https://i.imgur.com/H4MtKsd.png)

這種架構下可以組合出一個有趣的玩法，譬如第一個CNI使用常見的 CNI，第二個 CNI 則使用搭配 Static IPAM 的方式來設定固定 IP。
因此該 Pod 本身會有兩個網卡，其中一個是動態取得的IP，該 IP 會走 Control Plane 的網路，而第二個網卡則是設定固定 IP，該 IP 會走 Data Plane Network。

總結來說，引入 Metaplugin CNI 的架構，可以讓 k8s 的 CNI 變得彈性與靈活，不過前提是
**團隊針對網路有特別需求**，否則使用最簡單的 CNI 架構即可。

## 抽離 Kubernetes

有了這種混雜使用 CNI 的能力後，下一個思考的問題就是如何將 Pod 與現有的網路架構直接整合，能否跳脫 Kubernetes 的架構?

一種最簡單的做法就是使用 [SRIOV](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin) 這種網路模型，將支援的實體網卡(PF)拆分多個虛擬介面(VF)並且將這些虛擬介面直接送到 Pod 內使用。
這種架構下， Pod 內所有從該網卡出去的流量都會直接從節點上的網卡出去，不會受到節點本身 **Linux Kernel Network Stack** 眾多功能的影響，常見的 conntrack, iptables 都沒有辦法運作。 相對的該流量會直接流入到該實體網卡所對接的網路中，所有的流量控管與處理就可以於該網路中去處理，譬如 vlan, ACL 等。

下列這張圖展示了一種使用 SR-IOV 的架構
![](https://i.imgur.com/Ik4XkP0.png)

1. 環境中有兩種網路，所有 **對使用 data plane network 有興趣的 Pod** 都描述要使用 SR-IOV CNI。
2. SR-IOV CNI 可以搭配 **固定IP** 或是 **固定 MAC Address** 來使用，如果使用 **固定 MAC address** 的話也可以搭配 DHCP 來分配固定 IP。
3. Pod 創立後系統中就會產生兩張網卡，分別對應到 Control Plane 以及 Data Plane
4. 使用 Data Plane 的網卡由於流量不會經過節點本身的 Linux Kernel 處理，因此可以直接使用跟 data plane network 相同的網段來設定，不需要額外的 NAT 來轉換封包。


此架構下重新檢視最初的三個困境

1. 固定 IP/MAC
2. 主從網路
3. 多租戶隔離

透過 Metaplugin 的架構，前述兩個議題都有辦法完成，而第三個主要取決於想要隔離的強度，基本上釐清需求後，都有辦法透過實作出來。

但是 Metaplugin CNI 也並非完美，因為 Pod 擁有能力設定多個網卡與 IP，這個作法實際上與 Kubernetes 是完全不合的。
Kubernetes 預設是每個 Pod 只會有一個 IP，所以其內部的資料結構只會記錄一個 IP 地址，同時如果要使用 Kubernetes Service 時，到底該 Service 要使用哪個 IP 來傳輸?

如果使用的是 Mutlus 這套解決方案的話，第二張網卡以後的資訊都沒有辦法讓 K8s Service 使用，的但是如果使用的是 DANM 的話則提供了辦法解決，讓你可以繼續使用 k8s service 來訪問不同的網卡。

# 結論

Kubernetes 並非萬能，對於大多的應用來說網路一直都不是重點，能通互相存取即可。但是對部分產業與應用程式來說，網路則是一個硬需求，這種情況下就會發覺 Kubernetes 本身的不足。

Metaplugin CNI 的概念我認為大概99%的使用者都沒有這個需求，但是學習這種概念的設計可以幫助理解其他解決方案的設計，譬如 Kubevirt 專案底下就使用 Multus 來設定 VM 間的網路。

最後還是老話一句，網路好難



# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

詳細可以參閱
矽谷牛線上學院
https://course.hwchiu.com

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
