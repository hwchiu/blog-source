---
title: '[Kubernetes] DNS setting in your Pod'
keywords: 'kubernetes,security,Linux,k8s,TLS'
tags:
  - Kubernetes
  - Linux
  - DNS
abbrlink: 36819
date: 2018-08-03 10:52:35
description:
---

在`Kubernetes` 裡面看到 `DNS` 這個字眼，實際上可以想到非常多相關的元件與功能，
譬如用來提供 `kubernetes` 集群內服務的 `kube-DNS`, 或是透過 `kubernetes service` 產生之獨一無二的 FQDN 名稱，最後就是本篇文章想要分享的一個元件， `Pod` 內的 `DNS` 設定。

在大部分的使用情境之下，通常不會去關心這個部分，直接套用 `kubernetes` 預設的設定幫忙把相關的給處理完畢，然而隨者 `kubernetes` 愈來愈熱門，使用的情境愈來愈多。
譬如跟 NFV(Network Function Virtualization) 有關的應用中，我們所運行的任何應用程式 (Pod) 可能就會有 `DNS` 相關的需求。

接下來我們使用下面這張架構圖來說明可能的使用情境
![NFV Case](https://i.imgur.com/DcFCnLw.png)
<!--more-->

## Introduction
### The Reason Why We Need This
一般的使用情境下，我們的`kubernetes` 的集群使用方式就如同圖片中紫色/粉紅色(Pod3)區塊一樣，所有的 `Pod` 如果有任何要存取`DNS`的需求，都會透過集群內`K8S-DNS`來處理對應的請求與回覆。

然而在 NFV 的使用情境下，網路變成一個很重要的區塊，整體的效能都取決於該應用程式的設計與整個及集群的網路架構設計。

這部分的應用程式通常都會追求高輸出或是低延遲，同時也要避免這些流量會跟其他無關的流量使用相同的網路線才傳輸，造成該應用程式沒有辦法得到最好的效能。

在這種情況下，通常整個集群就會將網路設計成兩種架構，分別是`Control Network`, `Data Network` 兩個完全不同用途的網路架構。

在 `Kubernetes` 的架構下， `Control Network` 就類似圖示中的 `Cluster Network`，負責整個集群之間的溝通。
對應到圖中綠色/橘色(Pod1,Pod2)這兩個區塊則是所謂的 `Data Network`.
其網路卡本身也是獨立出來，不會與本來的 `kubernetes` 集群互相衝突，其之間的流量就透過獨立的網路傳輸。

獨立出來的網路架構就意味者，這些特殊應用的`Pod`基本上沒有辦法跟`Kubernetes`集群內的`Kube-DNS` 互連，網路本身就隔絕了這些連線。

此外，這些應用程式可能會在外部有自己的 `DNS Server` 來使用，所以在這種類型下，我們會希望這些應用程式 (Pod2/Pod3) 能夠使用自定義的 `DNS Server` 來使用，而
並非集群內建的 `DNS Server` 。

### Pod

透過上述的介紹，已經可以大概了解我們的需求，希望能夠針對 `Pod` 內去進行 `DNS` 客製化的設定。

在目前 `kubernetes` v1.11.0 裡面，對於 `Pod` 來說，`DNS` 相關的選項有兩個，分別是 `DNSConfig` 以及 `DNSPolicy`。

其中 `DNSConfig` 代表的是客製化的 `DNS` 參數，而 `DNSPolicy` 則是要如何幫 `Pod`設定預設的 `DNS`。


接下來我們來看看對於 `Pod` 來說，到底有哪些 `DNS` 的設定可以使用

**底下範例使用的全部的 `kubernetes yaml` 檔案都可以在 [Hwchiu KubeDemo](https://github.com/hwchiu/kubeDemo/tree/master/dns) 找到**
## DNSConfig
`DNSConfig` 意味可以讓操作者延伸當前 `Pod` 內關於 `DNS` 的設定，這邊要特別注意的是，我使用的字眼是 `延伸` 而非 `設定`，這是因為透過下個章節的 `DNSPOlicy`, 每個 `Pod` 都會有一組預設的 `DNS` 設定。
透過 `DNSConfig` 我們可以繼續往上疊加相關的 `DNS` 參數到 `Pod` 之中。
目前總共支援三個參數可以設定，分別是

1. nameservers: 
2. searches:
3. options:

這三個參數其實就是對應到大家熟悉的 `/etc/resolv.conf` 裡面的三個參數，這邊就不針對 `DNS` 進行介紹，不熟悉的朋友可以自行在去 **Google** 學一下這些參數。

在 `Kubernetes`裡面，這三個變數都歸屬於 `dnsConfig` 下面，而 `dnsConfig` 則是歸屬於 `PodSpec` 底下，因為 `Pod` 內所有的 `Container` 都共享相同的 `Network Namespace`, 所以網路相關的設定都會共享。

這邊提供一個簡單的 `yaml` 範例，也可以在 [這邊找到](https://github.com/hwchiu/kubeDemo/blob/master/dns/dnsSetting/ubuntu.yml)

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-setting
  namespace: default
spec:
  containers:
  - image: hwchiu/netutils
    command:
      - sleep
      - "360000"
    imagePullPolicy: IfNotPresent
    name: ubuntu
  restartPolicy: Always
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

在部屬上述 `yaml` 檔案後，透過下列指令可以觀察到系統上面 `DNS` 相關的設定變得非常多。
```shel=
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl exec ubuntu-setting cat /etc/resolv.conf
```
```shell=+
nameserver 10.96.0.10
nameserver 1.2.3.4
search default.svc.cluster.local svc.cluster.local cluster.local ns1.svc.cluster.local my.dns.search.suffix
options c
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$
```

針對 `nameserver` 可以觀察到多了 **1.2.3.4**,
而 `search` 則是多了 **ns1.svc.cluster.local my.dns.search.suffix** 這兩個自定義的數值，最後 `options` 則增加了我們範例中的 **ndots:2 edns0**

`DNSConfig` 非常簡單直覺，如果你有自己需要的 `DNS` 參數需要使用，就可以透過這個欄位來設定。

## DNSSPolicy
前面提過， `DNSConfig` 提供的是延伸 `Pod` 內預設的 `DNS` 設定，而 `DNSPolicy` 就是決定 `Pod` 內預設的 `DNS` 設定有哪些。
目前總共有四個類型可以選擇
- None
- Default
- ClusterFirst
- ClusterFirstHostNet

接下來針對這四個選項分別介紹

### None
`None` 的意思就如同字面上一樣，將會清除 `Pod` 預設的 `DNS` 設定，於此狀況下， `Kubernetes` 不會幫使用者的 `Pod` 預先載入任何自身邏輯判斷得到的 `DNS` 設定。
但是為了避免一個 `Pod` 裡面沒有任何的 `DNS` 設定存在，因此若使用這個 `None`的話，則一定要設定 `DNSConfig` 來描述自定義的 `DNS` 參數。

使用[下列 Yaml](https://github.com/hwchiu/kubeDemo/blob/master/dns/None/ubuntu.yml) 檔案來進行測試

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-none
  namespace: default
spec:
  containers:
  - image: hwchiu/netutils
    command:
      - sleep
      - "360000"
    imagePullPolicy: IfNotPresent
    name: ubuntu
  restartPolicy: Always
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

部屬完畢後，透過下列指令觀察該 `Pod` 內的 `DNS` 設定，可以觀察到跟之前 `DNSConfig` 的結果有一點差異，這時候只有顯示我們在 `Yaml` 裡面所設定的那些資訊，集群本身預設則不會幫忙加入任何 `DNS` 了。

```shell=
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl exec ubuntu-none cat /etc/resolv.conf
nameserver 1.2.3.4
search ns1.svc.cluster.local my.dns.search.suffix
options ndots:2 edns0
```

### Default
`Default` 代表的是希望 `Pod` 裡面的 `DNS` 設定請繼承運行該 `Pod` 的 `Node` 上的 `DNS` 設定。
簡單來說就是，該 `Pod` 的 `DNS` 設定會跟節點機器完全一致。

接下來使用[下列Yaml](https://github.com/hwchiu/kubeDemo/blob/master/dns/default/ubuntu.yml)檔案ˋ來進行測試

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-default
  namespace: default
spec:
  containers:
  - image: hwchiu/netutils
    command:
      - sleep
      - "360000"
    imagePullPolicy: IfNotPresent
    name: ubuntu
  restartPolicy: Always
  dnsPolicy: Default
```

首先，我們先觀察本機上面的 `DNS` 設定，這邊因為我的 `kubernetes` 集群只有一台，所以我可以確保該 `Pod` 一定會運行在我這台機器上。

```shell=
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ cat /etc/resolv.conf
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.0.2.3
```
這時可以觀察到，機器上本來的`DNS`設定非常簡單，只有單純的 `10.0.2.3`。
接下來我們觀察該 `Pod` 內的 `DNS` 設定
```shell=+
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl exec ubuntu-default cat /etc/resolv.conf
nameserver 10.0.2.3
```

可以看到這兩個的 `DNS` 設定是完全一致的，該 `Pod`內的 `DNS` 設定已經直接繼承自該運行節點上的設定了。

### ClusterFirst
相對於上述的 `Default` 設定， `ClusterFisrt` 是完全相反的操作，會預先把 `kube-dns` 的資訊當做預設參數寫入到該 `Pod` 內的 `DNS` 設定。


>特別注意的是， ClusterFirst 是預設的行為，若沒有在 Pod 內特別描述 PodPolicy, 則預設會以 ClusterFirst 來看待


接下來使用[下列Yaml檔案](https://github.com/hwchiu/kubeDemo/blob/master/dns/ClusterFirst/ubuntu.yml)來觀察結果看

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-clusterfirst
  namespace: default
spec:
  containers:
  - image: hwchiu/netutils
    command:
      - sleep
      - "360000"
    imagePullPolicy: IfNotPresent
    name: ubuntu
  restartPolicy: Always
  dnsPolicy: ClusterFirst
```

首先，因為 `ClusterFirst` 使用的是 `kube-dns` 的 `clusterIP` 作為 `DNS` 的位置，所以我們先透過下列指令觀察 `kube-dns` 的 IP 位置

```shell=
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl -n kube-system get svc kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   19h
```

根據上述結果可以觀察到，當前 `kube-dns` 使用的 `clusterIP` 是 `10.96.0.10`。

接下來去觀察對應 `Pod` 內的 `DNS` 設定
```shell=+
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl exec ubuntu-clusterfirst cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$
```

這邊可以看到一些資訊
1. nameserver 對應到的就是 `kube-dns` 的 `clusterIP`
2. search 這邊則會產生不少個關於 `kubernetes` 集群內使用的規則，這邊要特定注意到 `default` 這個數值會隨者該 `Pod` 所屬的 `namespace` 有所改變
> 在 `kubernetes` 中，每個 service 都會打開一個 `$service`.`$namespace`.svc.cluster.local 的 `DNS`.
> 對於 `Pod` 來說, 相同 `namespace` 可以直接用 `$service` 直接查詢到，若是不同的 `namespace`則要輸入 `$service.$namespace`
> 這邊就是依賴這些 search 來達成這些功能的


此外，`ClusterFirst` 有一個衝突，如果你的 `Pod` 有設定 `HostNetwork=true` 的話，則 `ClusterFirst` 就會變成 `Default` 來使用。

#### HostNetwork
使用[下列 Yaml](https://github.com/hwchiu/kubeDemo/blob/master/dns/hostnetwork/ubuntu-default.yml) 來觀察測試一下
```yam=
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-hostnetwork-policy-default
  namespace: default
spec:
  containers:
  - image: hwchiu/netutils
    command:
      - sleep
      - "360000"
    imagePullPolicy: IfNotPresent
    name: ubuntu
  hostNetwork: true
  restartPolicy: Always
  dnsPolicy: ClusterFirst
```

```shell=
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl exec ubuntu-hostnetwork-policy-default cat /etc/resolv.conf
nameserver 10.0.2.3
```

可以觀察到，這個情況下， `DNS` 的設定會被設定回節點上的設定。

這邊稍微來解釋一下這個設計上的原理以及流程
1. 因為設定 `HostNetwork=true`, 會讓該 `Pod` 與該節點共用相同的網路空間(網卡/路由等功能)
2. 預設的 `kube-dns` 是採取 `ClusterIP` 的 `kubernetes serivce`. 這種情況下，只有屬於 `Cluster` 內的 `Pod` 可以存取該 `ClusterIP`.
3. 所以設定 `HostNetwork=true` 的 `Pod` 就沒有辦法存取該 `ClusterIP`
4. 於是預設就會將對應的 `DNS` 設定改回 `Default` 的形式，從節點繼承其 `DNS` 設定

在這種情況下，就會有人想要問，我如果刻意的想要這樣設定不行嘛?
原先的設計中，是沒有辦法刻意處理的，原因是當 `Pod yaml` 檔案送出去後，在發現沒有設定 `PodPolicy` 的情況下，會自動幫你把該`PodPolicy` 補上 `ClusterFirst` 的數值。

然後最後面的程式處理邏輯中，其實並沒有辦法分別下列兩種情況
1. HostNetwork
> 我希望走 Host DNS
3. HostNetwork & PodPolicy=ClusterFirst.
> 我希望走 ClusterIP DNS

上述兩種情況對於後端的程式來看都長得一樣
`HostNetwork & PodPolicy=ClusterFirst.`
因此完全沒有辦法分辨

我們可以直接從 [Kubernetes 程式碼](https://github.com/kubernetes/kubernetes/blob/release-1.11/pkg/kubelet/network/dns/dns.go#L258) 來閱讀一下其運作流程
```go=
func getPodDNSType(pod *v1.Pod) (podDNSType, error) {
	dnsPolicy := pod.Spec.DNSPolicy
	switch dnsPolicy {
	case v1.DNSNone:
		if utilfeature.DefaultFeatureGate.Enabled(features.CustomPodDNS) {
			return podDNSNone, nil
		}
		// This should not happen as kube-apiserver should have rejected
		// setting dnsPolicy to DNSNone when feature gate is disabled.
		return podDNSCluster, fmt.Errorf(fmt.Sprintf("invalid DNSPolicy=%v: custom pod DNS is disabled", dnsPolicy))
	case v1.DNSClusterFirstWithHostNet:
		return podDNSCluster, nil
	case v1.DNSClusterFirst:
		if !kubecontainer.IsHostNetworkPod(pod) {
			return podDNSCluster, nil
		}
		// Fallback to DNSDefault for pod on hostnetowrk.
		fallthrough
	case v1.DNSDefault:
		return podDNSHost, nil
	}
	// This should not happen as kube-apiserver should have rejected
	// invalid dnsPolicy.
	return podDNSCluster, fmt.Errorf(fmt.Sprintf("invalid DNSPolicy=%v", dnsPolicy))
}
```

這邊可以看到一旦是 `DNSClusterFirst` 的情況下，若有設定 `HostNetwork`, 最後就會直節回傳 `podDNSHost` 節點的 `DNS` 設定回去。

為了解決上述的問題，所以引進了一個新的型態 `ClusterFirstHostNet`

### ClusterFirstHostNet
`ClusterFirstHostNet` 用途非常簡單，我希望滿足使用 `HostNetwork` 同時使用 `kube-dns` 作為我 `Pod` 預設 `DNS` 的設定。

根據上面的程式碼也可以觀察到
```go=
	case v1.DNSClusterFirstWithHostNet:
		return podDNSCluster, nil
```

其實只要將 `DNSPolicy` 設定為 `ClusterFirstHostNet`, 就會一律回傳 `kube-dns` 這種 `clusterIP` 的形式。

這邊我們使用 [下列 Yaml 檔案](https://github.com/hwchiu/kubeDemo/blob/master/dns/hostnetwork/ubuntu-dns.yml)

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-hostnetwork-policy
  namespace: default
spec:
  containers:
  - image: hwchiu/netutils
    command:
      - sleep
      - "360000"
    imagePullPolicy: IfNotPresent
    name: ubuntu
  hostNetwork: true
  restartPolicy: Always
  dnsPolicy: ClusterFirstWithHostNet
```

部屬完畢後執行下列指令觀察該 `Pod` 的狀態
```shell=
vagrant@vortex-dev:~/kubeDemo/dns/dnsSetting$ kubectl exec ubuntu-hostnetwork-policy cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

可以發現這時候的 `DNS` 設定就會使用的是 `ClusterIP` 的設定了。


## Summary
目前 Pod 有提供兩種設定 `DNS` 的參數來使用者來管理 `DNS` 相關的參數。

- DNSPolicy
- DNSConfig

### DNSPolicy
代表的是該 `Pod` 預設的 `DNS`  設定，目前總共有四種類型可以使用

**Default**
- 代表的是繼承自運行節點的 `DNS` 設定

**None** 
- 代表的是完全不設定任何 `DNS` 的資訊，所有的 `DNS` 都依賴 `DNSConfig` 欄位來描述 (若採用 None, 一定要設定 `DNSConfig`)

**ClusterFirst** 
- 代表是使用 `kube-dns` 提供的 `clusterIP` 作為預設的 `DNS` 設定
- 同時若使用者沒有在 `Pod` 內去描述 `DNSConfig` ，則預設會使用 `ClusterFirst` 這個設定

**ClusterFirstWithHostNet** 
- 這個選項就是特別針對 `HostNetwork=true` 創立的
- 可以提供使用節點網路但是同時又使用 `kube-dns` 提供的 `clusterIP` 作為其 `DNS` 的設定。

### DNSConfig
`DNSConfig` 可以讓使用者直接輸入 `DNS` 相關的參數，該參數會擴充該 `Pod`原本的 `DNS` 設定檔案。
