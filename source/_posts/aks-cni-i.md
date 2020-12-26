---
title: "Azure Kubernetes Service (AKS) - CNI (I)"
keywords: 'Kubernetes,Network,Azure,CNI'
abbrlink: 17245
date: 2019-03-26 00:32:42
tags:
  - Azure
  - Kubernetes
  - CNI
  - Network
description: 除了自行架設 Kubernetes 之外，採用公有雲廠商所提供的 Kubernetes Service 也是一個方便的選擇，然而這種情況下有許多的設定跟功能都會依賴該公有雲廠商自行實作，大部分的功能都會與公有雲本身的架構進行高度整合以提供更方便的使用與操作。本文針對 Container Network Interface (CNI) 於 Azure 中的實現與使用進行了討論，藉此了解公有雲的 CNI 有什麼特別的設計與使用方式

---

# Preface
除了自行架設 Kubernetes 之外，愈來愈多的公有雲提供商也都提供了 Kubernetes 的服務，從 Google 的 `GKE`(Google Kubernetes Engine), Amazon 的 `AKS`(Amazon Kubernetes Service) 到 Azure 的 `AKS`(Azure Kubernetes Service)，使用者都可以很簡易的透過其提供的操控介面創造一個全新的 `kubernetes` 集群來使用。

對我來說，這些公有雲集群與自建集群有非常多的差異，由於這些公有雲所提供的 `kubernetes` 都基於這些公有雲的基本架構上，所以有特別多的資源可以進行整合，譬如 `Load-Balancer/Ingress` 就可以搭配自家的 `LoadBalancer` 來使用，同時 `Ingress` 所使用的 `hostname` 也能夠跟相關的 `DNS` 整合。此外還有一個很明顯的差異就是 `IP` 的分配及使用。 不論是 `AKS` 或是 `EKS` 都提供了讓 `Node` 與 `Pod` 使用相同的 `IP/Subnet` 的機制，而這部分則是牽扯到 `Container Network Interface` 的設計，因此如 `AKS/EKS` 都有開源使用到的 `CNI`.

[Amazon-VPC-CNI-K8S](https://github.com/aws/amazon-vpc-cni-k8s)
[Azure-Container-Networking](https://github.com/Azure/azure-container-networking)

透過研究這些 `CNI` 的設計原理，可以學習到滿多的設計想法，同時若今天也有相同的需求，也可以作為一個參考對象去學習。

接下來會針對 `AKS` 的部分去探討其 `CNI` 的運作，並且分成兩篇文章來研究。第一篇文章主要會針對 `IP` 的管理，包含了分配與設定，以及相關的檔案資訊，第二篇文章則會針對 `Routing` 的設定，這種網路設定下，實際上封包再傳輸的時候有哪些不同於 `Flannel/Calico/Bridge` 的設定與用法。

# Introduction
正式探討 `Azure-Container-Networking CNI` 的運作之前，有一些基本的概念要先瞭解，當你透過 `Azure Portal` 創立一個三個節點的 `AKS` 服務後，`Azure` 會幫你創建不少相關的資源, 其中跟本文關係較重要的幾個分別是
1. Nodes (Virtual Machines)
2. VNet (Virtual Networks)
3. Kubernetes

Nodes 的數量就是 `AKS` 的節點數量，而節點上能夠擁有的 `IP/Subnet` 則都是在 `VNet` 該資源中管理。 該 `Vnet` 所管理的 `IP/Subnet` 不但跟節點有關，也跟運行在該節點上的 `Pod` 有關，因為 `Pod` 最終使用的 `IP/Subnet` 都是在 `VNet` 這邊去設定與控管。
而這些 Nodes 之間的網路連線則是 `Azure` 本身幫忙處理的，實際上是透過 `Azure SDN Fabric` 的概念來幫忙處理這些節點間的網路傳輸，包含相關的 `Virtual Networks` 的處理。
最後則是基於這些架構下於節點上安裝 `kubernetes` 相關的元件並提供一個可操作的 `kubernetes` 叢集。

本文的所有環境都基於下列的 `AKS` 架構圖，如下圖。

![Imgur](https://i.imgur.com/dsrygTc.png)
![Imgur](https://i.imgur.com/N1a8mEk.png)



此外可以參考這篇[官方文章學習](https://docs.microsoft.com/en-us/azure/aks/ssh)如何透過 `SSH` 連線到 `AKS` 的節點中來進行本文的研究


# Azure-Container-Networking(CNI)
`Container Network Interface CNI` 是一套規範，用來處理容器本身的網路設定，至於要提供什麼樣的網路能力本身並沒有定義，而是根據不同的 `CNI` 實現自行決定，根據不同的情境與設定來提供不同的功能與效果。

`CNI` 本身的運作原理是以機器為單位，不以叢集為單位，因此每台機器都要安裝欲使用的 `CNI` 執行檔以及對應的設定檔案。
以本文的範例為例，三個節點就意味三個節點上都會安裝對應的 `CNI` 執行檔，也就是 `Azure-Container-Networking` 專案的產物，同時也會有對應的設定檔案。


我們可以於 `AKS` 的節點上面的 `/opt/cni/bin` 中觀察到目前系統中安裝的 `CNI` 執行檔

```bash=
azureuser@aks-agentpool-15026905-1:~$ ls /opt/cni/bin/
azure-vnet       bridge  flannel      host-local  loopback  portmap  sample  vlan
azure-vnet-ipam  dhcp    host-device  ipvlan      macvlan   ptp      tuning
```

其中 `azure-vnet`, `azure-vnet-ipam` 兩個則是 `Azure-Container-Networking` 這個專案所產生的，一個用來處理網路連接，一個用來處理 `IP/Subnet` 的取得與設定

```bash=
azureuser@aks-agentpool-15026905-1:~$ ls /etc/cni/net.d/
10-azure.conflist
azureuser@aks-agentpool-15026905-1:~$ sudo cat /etc/cni/net.d/10-azure.conflist
{
   "cniVersion":"0.3.0",
   "name":"azure",
   "plugins":[
      {
         "type":"azure-vnet",
         "mode":"bridge",
         "bridge":"azure0",
         "ipam":{
            "type":"azure-vnet-ipam"
         }
      },
      {
         "type":"portmap",
         "capabilities":{
            "portMappings":true
         },
         "snat":true
      }
   ]
}
```

這個設定檔案的內容我們先暫時不去解讀，只要知道每一台 `AKS` 節點上都會有一個獨立的檔案來負責該節點的 `CNI` 即可。

`CNI` 的運作流程簡單來說可以分成多個步驟，分別是
1. Kubelet 收到通知要創建對應的 Pod
2. Kubelet 透過 `CRI` 創建 `Pause Container`
3. Kubelet 呼叫 `CNI` 並且將 `Pause Container` 的部份資訊當作參數傳給 `CNI`
4. `CNI` 根據自己的實現為 `Pause Container` 提供網路能力
5. Kubelet 相信 `CNI` 已經完成所需任務，接下來創建使用者需要的 `Containers`，這些 `Containers` 的網路空間都直接掛載到 `Pause Container` 上

而本文主要探討的 `Azure-Vnet CNI` 主要負責的部分就是 **4** 所描述的工作，為 `Pause Container` 提供網路能力。

此外對於 CNI 這個議題有興趣的人可以參閱下列系列文章來學習更多 CNI 相關的資訊
1. [常見 CNI (Container Network Interface) Plugin 介紹](https://www.hwchiu.com/cni-compare.html)
2. [[Container Network Interface] CNI Introduction](https://www.hwchiu.com/introduce-cni-ii.html)
3. [[Container Network Interface] Bridge Network In Docker](https://www.hwchiu.com/introduce-cni-i.html)
4. [[Container Network Interface] Write a CNI Plugin By Golang](https://www.hwchiu.com/introduce-cni-iii.html)
5. [CNI 常見問題整理]((https://www.hwchiu.com/cni-questions.html)

## Azure-VNET
`Azure-VNET CNI` 的詳細原始碼都在 [Github-Azure-Container-Networking](https://github.com/Azure/azure-container-networking/tree/master/cni), 有興趣的讀者可以自行閱讀來學習。

基本上 `Azure-VNET` 跟常見的 `L2 Linux Bridge` 非常相似，其運作流程如下
1. 創建一個 Linux Bridge `Azure0` (若存在就不創造)
2. 創造一條 `Veth` 的 `Linux Logical Link`
3. 將該 `Veth` 的一端放到 `Pause Container` 內，並且命名為 `eth0`.
4. 將該 `Veth` 的另一端綁到 `Azure0` 上，該 `Veth` 的名稱都是會 `azvxxxxxxx`
5. 呼叫 `Azure-VNET-IPAM` 去取得可用的 `IP/Subnet` 並且設定到 `Pause Container` 內的 `eth0` 介面
    - 這部分會在下個章節解釋其運作



有興趣的讀者也可以參閱 [AKS SSH to Node](https://docs.microsoft.com/en-us/azure/aks/ssh) 這篇文章的方式連接到 `AKS` 內部的節點來實際看看這些資訊

我們可以使用 `brctl` 這個工具來觀察 `Linux Bridge` 的關係

```bash=
azureuser@aks-agentpool-15026905-1:~$ brctl show
bridge name     bridge id               STP enabled     interfaces
azure0          8000.000d3a51cdbb       no              azv1be2dcd6c83
                                                        azv2705efbd6d8
                                                        azv499967b4ec4
                                                        azv4d966079c93
                                                        azv5a6822c76a3
                                                        azv93928864e55
                                                        azva985f33c456
                                                        azvaa217c54e39
                                                        azvbc5198bad97
                                                        azvc1ab312d517
                                                        azvc92e6588502
                                                        eth0
docker0         8000.024220b0b010       no
```

基本上整個節點中的狀況如下圖，每個 `Pod` 都會透過 `Veth` 與 `Azure0 ` 這個 `Linux Bridge`相連
![Imgur](https://i.imgur.com/DSzvmqc.png)




基本上該節點上面有多少個沒有設定 `HostNetwork=true` 的 `Pod`, 就會有多少條對應的 `azvxxxx veth link`.

此外，我個人對於 `Azure-VNET CNI` 覺得很好也喜愛的地方就是留有大量地資訊在節點上，這部分不論是對於研究或是除錯都非常的好用。

該資訊被放置於 `/var/run/azure-vnet.json`, 當 `azure-vnet CNI` 每次被呼叫來執行對應工作的時候，都會詳細的紀錄這次的資訊，包含 `ContainerID`, `PodName`, `IP`, `Route`等各式各樣的設定資訊。


```json=
{
 "Network": {
         "Version": "v1.0.17",
         "TimeStamp": "2019-03-22T14:30:27.934886517Z",
         "ExternalInterfaces": {
                 "eth0": {
                         "Name": "eth0",
                         "Networks": {
                                 "azure": {
                                         "Id": "azure",
                                         "Mode": "bridge",
                                         "VlanId": 0,
                                         "Subnets": [
                                                 {
                                                         "Family": 2,
                                                         "Prefix": {
                                                                 "IP": "10.240.0.0",
                                                                 "Mask": "//8AAA=="
                                                         },
                                                         "Gateway": "10.240.0.1"
                                                 }
                                         ],
                                         "Endpoints": {
                                                 "1bd38ad8-eth0": {
                                                         "Id": "1bd38ad8-eth0",
                                                         "SandboxKey": "",
                                                         "IfName": "eth0",
                                                         "HostIfName": "azv5a6822c76a3",
                                                         "MacAddress": "nkfKrRbG",
                                                         "InfraVnetIP": {
                                                                 "IP": "",
                                                                 "Mask": null
                                                         },
                                                         "IPAddresses": [
                                                                 {
                                                                         "IP": "10.240.0.94",
                                                                         "Mask": "//8AAA=="
                                                                 }
                                                         ],
                                                         "Gateways": [
                                                                 "10.240.0.1"
                                                         ],
                                                         "DNS": {
                                                                 "Suffix": "",
                                                                 "Servers": [
                                                                         "168.63.129.16"
                                                                 ]
                                                         },
                                                         "Routes": [
                                                                 {
                                                                         "Dst": {
                                                                                 "IP": "0.0.0.0",
                                                                                 "Mask": "AAAAAA=="
                                                                         },
																		 "Src": "",
                                                                         "Gw": "10.240.0.1",
                                                                         "Protocol": 0,
                                                                         "DevName": "",
                                                                         "Scope": "0"}
                                                         ],
                                                         "VlanID": 0,
                                                         "EnableSnatOnHost": false,
                                                         "EnableInfraVnet": false,
                                                         "EnableMultitenancy": false,
                                                         "NetworkNameSpace": "/proc/5336/ns/net",
                                                         "ContainerID": "1bd38ad8d840dd1f84597d4343b3bd116188cd1e4a797cc31bdc1aa3dc654a5b",
                                                         "PODName": "addon-http-application-routing-external-dns-74db4f974b-8w4wz",
                                                         "PODNameSpace": "kube-system"
                                                 },
..................................
```

上述 `Endpoints` 裡面的每個物件都會描述到一個 `Pause Container` 的網路環境與設定，不過對於 `CNI` 來說本身不在意是不是 `Pause Container`. 這部分是 `Kubelet` 自行實現的邏輯，所以基本上不會再 `CNI` 這邊看到 `Pause Container` 相關的文字。


## Azure-VNET-IPAM
接下來要探討的是要如何分配 `IP/Subnet` 這件事情，基本上任意兩個 `Pod` 都不應該使用相同的 `IP/Subnet`。但是對於 `CNI` 這種非中央極權管理的執行程式來說，要做到不衝突就必須要有一些機制了。
各式各樣的 `CNI` 都有自己的機制來處理，不論是透過 `ectd` 或是自行實現集中式管理機制，只要能夠避免分配重複 `IP/Subnet` 即可

前面有提過，三大公有雲的 Kubernetes Service 相較於自建來說有更多的優勢與特色就是因為可以將 KUbernetes 與公有雲內的設施與狀態結合來提供更多的功能

`Azure-VNET-IPAM` 這個 `IP/Subnet` 管理機制基本上就是與 `Azure` 的環境有高度整合，接下來我們來看一下其運作的原理。

每台機器上的 `Azure-VNET-IPAM` 這個 `CNI` 都會執行相同的運作原理，最後卻要取得不同的 `IP/Subnet`, 這整個運作原理如下
1. 每個 `Azure-VNET-IPAM CNI` 都會透過 `HTTP` 去詢問叢集 API, 來確認當前節點在 `VNET` 內可以擁有的 `IP` 數量
2. 從可用的 `IP` 數量內隨機挑選一個 `IP` 並返回
3. 最後 `Azure-VNET` 就會取得 `Azure-VNET-IPAM` 得到的 `IP/Subnet` 並且設定到對應的 `Pause Container` 裡面


## API
從[原始碼](https://github.com/Azure/azure-container-networking/blob/67debca9016218981b69ff954536418ea0903305/ipam/azure.go#L17-L23)中可以觀察到下列 URL 的設定
```go=
const (
	// Host URL to query.
	azureQueryUrl = "http://168.63.129.16/machine/plugins?comp=nmagent&type=getinterfaceinfov1"

	// Minimum time interval between consecutive queries.
	azureQueryInterval = 10 * time.Second
)
```

然而目前實際上使用沒有通，原因在於先前的 [Commit](https://github.com/Azure/azure-container-networking/commit/e5f6b0d03c96b3eab4ce76178ddb6d3d57fb92c9#diff-ce5c6e7b3ab2a2f573913afae54abd4b) 有修改過該 URL 的數值，而且該 Commit 距離這篇文章不到一個月前，所以我認為 Azure 上面還沒有採用新的版本，因此實際上操作時還是使用舊有的 URL。
```
Update host machine ip (#300)

* Limiting the size of our buffered payload to ~2MB

* Changing IPs for calls to host machines from 169.254.169.254 to 168.63.129.16.
```

根據上述的原始碼，我們可以直接在 `AKS` 節點中直接透過 `curl` 的方式去詢問，結果如下
```xml=
azureuser@aks-agentpool-15026905-0:~$ curl "http://169.254.169.254/machine/plugins?comp=nmagent&type=getinterfaceinfov1"
<Interfaces><Interface MacAddress="000D3A51C490" IsPrimary="true"><IPSubnet Prefix="10.240.0.0/16">
<IPAddress Address="10.240.0.35" IsPrimary="true"/>
<IPAddress Address="10.240.0.36" IsPrimary="false"/>
<IPAddress Address="10.240.0.37" IsPrimary="false"/>
<IPAddress Address="10.240.0.38" IsPrimary="false"/>
<IPAddress Address="10.240.0.39" IsPrimary="false"/>
<IPAddress Address="10.240.0.40" IsPrimary="false"/>
<IPAddress Address="10.240.0.41" IsPrimary="false"/>
<IPAddress Address="10.240.0.42" IsPrimary="false"/>
<IPAddress Address="10.240.0.43" IsPrimary="false"/>
<IPAddress Address="10.240.0.44" IsPrimary="false"/>
<IPAddress Address="10.240.0.45" IsPrimary="false"/>
<IPAddress Address="10.240.0.46" IsPrimary="false"/>
<IPAddress Address="10.240.0.47" IsPrimary="false"/>
<IPAddress Address="10.240.0.48" IsPrimary="false"/>
<IPAddress Address="10.240.0.49" IsPrimary="false"/>
<IPAddress Address="10.240.0.50" IsPrimary="false"/>
<IPAddress Address="10.240.0.51" IsPrimary="false"/>
<IPAddress Address="10.240.0.52" IsPrimary="false"/>
<IPAddress Address="10.240.0.53" IsPrimary="false"/>
<IPAddress Address="10.240.0.54" IsPrimary="false"/>
<IPAddress Address="10.240.0.55" IsPrimary="false"/>
<IPAddress Address="10.240.0.56" IsPrimary="false"/>
<IPAddress Address="10.240.0.57" IsPrimary="false"/>
<IPAddress Address="10.240.0.58" IsPrimary="false"/>
<IPAddress Address="10.240.0.59" IsPrimary="false"/>
<IPAddress Address="10.240.0.60" IsPrimary="false"/>
<IPAddress Address="10.240.0.61" IsPrimary="false"/>
<IPAddress Address="10.240.0.62" IsPrimary="false"/>
<IPAddress Address="10.240.0.63" IsPrimary="false"/>
<IPAddress Address="10.240.0.64" IsPrimary="false"/>
<IPAddress Address="10.240.0.65" IsPrimary="false"/>
</IPSubnet></Interface></Interfaces>
```

有趣的是，不同的 `AKS` 節點問到的結果會是不同的，所以每台節點上得 `AKS-VNET-IPAM` 都可以透過這個方式來取得該台節點上所擁有能夠使用的 `IP` 數量以及對應的網段。

此外 `AKS-VNET-IPAM` 也會在本機端記錄相對應的資訊, 檔案位置位於 `/var/run/azure-vnet-ipam.json`

其內容主要會紀錄本機端可以使用的所有 `IP 位置`
```json=
{
        "IPAM": {
                "Version": "v1.0.17",
                "TimeStamp": "2019-03-18T21:48:15.199533623Z",
                "AddressSpaces": {
                        "local": {
                                "Id": "local",
                                "Scope": 0,
                                "Pools": {
                                        "10.240.0.0/16": {
                                                "Id": "10.240.0.0/16",
                                                "IfName": "eth0",
                                                "Subnet": {
                                                        "IP": "10.240.0.0",
                                                        "Mask": "//8AAA=="
                                                },
                                                "Gateway": "10.240.0.1",
                                                                                              "Addresses": {
                                                        "10.240.0.36": {
                                                                "ID": "",
                                                                "Addr": "10.240.0.36",
                                                                "InUse": true
                                                        },
                                                        "10.240.0.37": {
                                                                "ID": "",
                                                                "Addr": "10.240.0.37",
                                                                "InUse": true
                                                        },
                                                        "10.240.0.38": {
                                                                "ID": "",
                                                                "Addr": "10.240.0.38",
                                                                "InUse": true
                                                        },
                                                        "10.240.0.39": {
                                                                "ID": "",
                                                                "Addr": "10.240.0.39",
                                                                "InUse": false
                                                        },
                                                        ..............
                                                        }}

```

基本上這些 `IP` 的資訊就跟 `Azure Portal` 裡面 `VNet (Virtual Networks)` 顯示的數量是一致的，因此我們可以透過 `Portal` 的方式就知道每台節點上運行的 `Pod` 的 `IP` 範圍。

## Capacity
一個很有趣的問題這時候就會浮現了，因為每個節點上能夠使用的 `IP` 數量跟 `Virtual Networks` 裡面的每台節點有關，那如果我部署大量的 `Pod` 到該節點上是否會發生 `IP` 不足夠的問題?

針對這個問題，我嘗試部署了超過 `IP` 數量的 `Pod` 到節點上，結果最後看到的是滿滿的 `Pending`
```
hwchiu-utils-785f896cc5-cgjsq   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-4gmwv   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-brnl6   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-gwgcz   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-v5s45   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-knz2r   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-26pn2   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-xcmft   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-wlckh   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-dm49b   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-nkcgw   0/1       Pending   0         4m
hwchiu-utils-785f896cc5-cwkbl   0/1       Pending   0         4m
```

這意味者這些 `Pod` 在 `kubernetes scheduler/kubelet` 這階段就被阻止了，根本沒有機會讓 `CNI` 繼續往下執行，這時候我們可以研究一下該 `kubernetes node` 的設定, 可以發現到有趣的資訊
```
Capacity:
 attachable-volumes-azure-disk:  8
 cpu:                            2
 ephemeral-storage:              30428648Ki
 hugepages-1Gi:                  0
 hugepages-2Mi:                  0
 memory:                         7137108Ki
 pods:                           30
Allocatable:
 attachable-volumes-azure-disk:  8
 cpu:                            1931m
 ephemeral-storage:              28043041951
 hugepages-1Gi:                  0
 hugepages-2Mi:                  0
 memory:                         5357396Ki
 pods:                           30
 ```

這兩者的資訊可以參閱 [reserve-compute-resources](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)

其中可以注意到 `Pod` 的數量被設定成 `30`，這個數字跟我們先前透過 `API` 去問到的 `IP` 數目幾乎是一致的
API 問到的 IP 還包括節點本身，所以是 30 + 1 = 31

所以這邊可以看到針對 `IP` 用光的問題， `AKS` 處理的方式除了 `CNI` 本身顯示錯誤訊息之外，最上層還先透過 `Kubernetes Node Capacity` 的方式進行了第一層的阻擋，理論上 `CNI` 本身不應該遇到 `IP` 用光的情形，因為根本不應該有超過數量的 `Pod` 被嘗試部署到該節點上。


# Summary
本文中我們研究了 `AKS` 中節點的 `CNI` 運作流程，包含了簡單的網路設定 (L2 Bridge)，同時也探討一下了 `IPAM` 的運作邏輯與流程，發現到該 `IPAM` 與 `Azure` 的架構本身有強烈的整合，透過 `Azure VNET` 的設定來控管每個節點上的 `Pod` 能夠使用的 `IP` 範圍與數量。

最後我們用一張流程圖來幫本文的 `CNI` 做一個總結
![Imgur](https://i.imgur.com/robQsi1.png)

下一篇文章會著重在此基礎上，當上述的 `L2 Bridge` 與 `IPAM` 都處理完畢後，這些 `Pod` 彼此中間是怎麼溝通的，不論是同節點或是跨節點的傳輸，特別是這些節點實際上都是 `VM` 的情況下，是要如何做到跨節點傳輸的

# Reference
- [Connect with SSH to Azure Kubernetes Service (AKS) cluster nodes](https://docs.microsoft.com/en-us/azure/aks/ssh)
- [Github-Azure-Container-Networking](https://github.com/Azure/azure-container-networking/tree/master/cni)
- [reserve-compute-resources](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=LG28Q5G

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