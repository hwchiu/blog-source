---
title: Kubernetes - IP 重複奇遇記
keywords: 'Kubernetes,Network,CNI, IPAM, Restore'
tags:
  - CNI
  - Kubernetes
  - Network
  - Rancher
description: >-
  本文探討一種備份復原過程中可能會發生的 IP 重複問題，文章開頭先用簡單的模擬方式來模擬如何產生 IP 重複問題，接下來針對 CNI
  的運作來探討其運作流程。
abbrlink: 19783
date: 2020-11-26 23:15:00
---

最近遇到一個有趣的案例，該案例中，新產生的 Pod 獲得的 IP 地址會跟已經存在的 Pod 相同。
最後就會看到叢集中有兩個 Pod 擁有相同的 IP 地址，進而導致衝突然後服務不能正常存取

這個情況並不常見，但是一旦發生時，並不是很容易可以釐清這個問題，因為 Kubernetes 本身並不負責這些 IP 的發放，而是透過 CNI 來處理，而 Kubelet 則是 CNI 與 Kubernetes 中間的溝通者。


本篇文章會嘗試模擬一個環境來探討這個問題，並且釐清問題的本質，當一切謎團都明瞭之時，我們再來看看真實情況下，有什麼踩到這個問題的可能性



# 模擬環境
本環境會採用 [KIND](https://kind.sigs.k8s.io/) 搭建一個多節點的 Kubernetes 叢集，使用的版本與相關設定檔案如下
```
→ kind --version
kind version 0.9.0

→ cat kind.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb
- role: worker
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb
- role: worker
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb
- role: worker
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb

→ kubectl version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.4", GitCommit:"d360454c9bcd1634cf4cc52d1867af5491dc9c5f", GitTreeState:"clean", BuildDate:"2020-11-11T13:17:17Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.8", GitCommit:"9f2892aab98fe339f3bd70e3c470144299398ace", GitTreeState:"clean", BuildDate:"2020-09-14T07:44:34Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}

→ kubectl get nodes
NAME                 STATUS   ROLES    AGE   VERSION
kind-control-plane   Ready    master   18m   v1.18.8
kind-worker          Ready    <none>   17m   v1.18.8
kind-worker2         Ready    <none>   17m   v1.18.8
kind-worker3         Ready    <none>   17m   v1.18.8

```

我們會透過一個 deployment 來模擬大量的 pod，該內容如下
```bash=
→ cat debug.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug-pod
spec:
  replicas: 30
  selector:
    matchLabels:
      app: debug-pod
  template:
    metadata:
      labels:
        app: debug-pod
    spec:
      containers:
        - name: debug-pod
          image: hwchiu/netutils:latest
```

# 模擬步驟

要模擬這個問題不會太困難，分成幾個步驟
1. 部署 pods
2. 停止節點上的 kubelet 並刪除節點上特定資料夾
3. 重啟節點上的 kubelet 並部署更多 pod 到該節點上

接下來我們就針對上述步驟來操作，並且以節點 `kind-worker3` 作為問題發生的節點

## 部署 Pods

```bash=
→ kubectl apply -f debug.yaml
deployment.apps/debug-pod created
```

這邊我們專注觀察 kind-worker3 上面所有 Pod 的 IP 即可
```bash=
 → kubectl get pods -o wide | grep kind-worker3
debug-pod-7f9c756577-6nv7b   1/1     Running   0          3m38s   10.244.2.6    kind-worker3   <none>           <none>
debug-pod-7f9c756577-9l766   1/1     Running   0          3m38s   10.244.2.8    kind-worker3   <none>           <none>
debug-pod-7f9c756577-bg89r   1/1     Running   0          3m39s   10.244.2.9    kind-worker3   <none>           <none>
debug-pod-7f9c756577-hb2cr   1/1     Running   0          3m38s   10.244.2.7    kind-worker3   <none>           <none>
debug-pod-7f9c756577-kzkjr   1/1     Running   0          3m39s   10.244.2.3    kind-worker3   <none>           <none>
debug-pod-7f9c756577-ncqg8   1/1     Running   0          3m38s   10.244.2.5    kind-worker3   <none>           <none>
debug-pod-7f9c756577-pwk9v   1/1     Running   0          3m39s   10.244.2.2    kind-worker3   <none>           <none>
debug-pod-7f9c756577-twwpn   1/1     Running   0          3m39s   10.244.2.4    kind-worker3   <none>           <none>
debug-pod-7f9c756577-z7kmq   1/1     Running   0          3m39s   10.244.2.11   kind-worker3   <none>           <none>
debug-pod-7f9c756577-zgs7k   1/1     Running   0          3m38s   10.244.2.10   kind-worker3   <none>           <none>
```

目前上面總共有 10 個 pod，所有 IP 都不一樣，非常正常


## 停止節點上的 kubelet 並刪除節點上特定資料夾
如同前面所述，我們針對 kind-worker3 進行模擬，因此執行下列指令來停止 `kubelet`，

```bash=
○ → sudo docker exec kind-worker3 systemctl stop kubelet
○ → sudo docker exec kind-worker3 systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
...
```

確認停止後，接者刪除相關資料夾
```bash=
sudo docker exec kind-worker3 rm -rf /run/cni-ipam-state
```

## 重啟節點上的 kubelet 並部署更多 pod 到該節點上
上述都完成後，我們重啟 kubelet

```bash=

○ → sudo docker exec kind-worker3 systemctl start kubelet
○ → sudo docker exec kind-worker3 systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) (thawing) since Fri 2020-11-27 04:04:28 UTC; 3s ago
       Docs: http://kubernetes.io/docs/
```

接下來我們要部署新的 pod 到 `kind-worker3` 身上，這邊我採取的方式是透過 **drain** 的概念，將 **kind-worker** 以及 **kind-worker2** 上面的 pod 都移除掉，然後重新部署到 **kind-worker3** 上，並且觀察這些 IP

```bash=
○ → kubectl drain kind-worker --ignore-daemonsets
○ → kubectl drain kind-worker2 --ignore-daemonsets
```

接者透過相同指令觀察 kind-worker3 上面的所有 pod IP

```bash=
→ kubectl get pods -o wide | grep kind-worker3 | sort -k 6
debug-pod-7f9c756577-hkhx9   1/1     Running   0          54s   10.244.2.10   kind-worker3   <none>           <none>
debug-pod-7f9c756577-zgs7k   1/1     Running   0          11m   10.244.2.10   kind-worker3   <none>           <none>
debug-pod-7f9c756577-qmjqk   1/1     Running   0          54s   10.244.2.11   kind-worker3   <none>           <none>
debug-pod-7f9c756577-z7kmq   1/1     Running   0          11m   10.244.2.11   kind-worker3   <none>           <none>
debug-pod-7f9c756577-pd6hl   1/1     Running   0          54s   10.244.2.12   kind-worker3   <none>           <none>
debug-pod-7f9c756577-ssmsv   1/1     Running   0          54s   10.244.2.13   kind-worker3   <none>           <none>
debug-pod-7f9c756577-bbx6j   1/1     Running   0          54s   10.244.2.14   kind-worker3   <none>           <none>
debug-pod-7f9c756577-9rn4j   1/1     Running   0          54s   10.244.2.15   kind-worker3   <none>           <none>
debug-pod-7f9c756577-sfwhs   1/1     Running   0          54s   10.244.2.16   kind-worker3   <none>           <none>
debug-pod-7f9c756577-jc8nw   1/1     Running   0          54s   10.244.2.17   kind-worker3   <none>           <none>
debug-pod-7f9c756577-5sb28   1/1     Running   0          54s   10.244.2.18   kind-worker3   <none>           <none>
debug-pod-7f9c756577-mgtzx   1/1     Running   0          54s   10.244.2.19   kind-worker3   <none>           <none>
debug-pod-7f9c756577-64xlq   1/1     Running   0          53s   10.244.2.20   kind-worker3   <none>           <none>
debug-pod-7f9c756577-jpmfk   1/1     Running   0          53s   10.244.2.21   kind-worker3   <none>           <none>
debug-pod-7f9c756577-pwk9v   1/1     Running   0          11m   10.244.2.2    kind-worker3   <none>           <none>
debug-pod-7f9c756577-rhfk4   1/1     Running   0          71s   10.244.2.2    kind-worker3   <none>           <none>
debug-pod-7f9c756577-kzkjr   1/1     Running   0          11m   10.244.2.3    kind-worker3   <none>           <none>
debug-pod-7f9c756577-njmw6   1/1     Running   0          71s   10.244.2.3    kind-worker3   <none>           <none>
debug-pod-7f9c756577-l9fl8   1/1     Running   0          71s   10.244.2.4    kind-worker3   <none>           <none>
debug-pod-7f9c756577-twwpn   1/1     Running   0          11m   10.244.2.4    kind-worker3   <none>           <none>
debug-pod-7f9c756577-ncqg8   1/1     Running   0          11m   10.244.2.5    kind-worker3   <none>           <none>
debug-pod-7f9c756577-rshx5   1/1     Running   0          71s   10.244.2.5    kind-worker3   <none>           <none>
debug-pod-7f9c756577-6nv7b   1/1     Running   0          11m   10.244.2.6    kind-worker3   <none>           <none>
debug-pod-7f9c756577-vklgs   1/1     Running   0          71s   10.244.2.6    kind-worker3   <none>           <none>
debug-pod-7f9c756577-hb2cr   1/1     Running   0          11m   10.244.2.7    kind-worker3   <none>           <none>
debug-pod-7f9c756577-jkpbd   1/1     Running   0          54s   10.244.2.7    kind-worker3   <none>           <none>
debug-pod-7f9c756577-9l766   1/1     Running   0          11m   10.244.2.8    kind-worker3   <none>           <none>
debug-pod-7f9c756577-lr8t5   1/1     Running   0          54s   10.244.2.8    kind-worker3   <none>           <none>
debug-pod-7f9c756577-bg89r   1/1     Running   0          11m   10.244.2.9    kind-worker3   <none>           <none>
debug-pod-7f9c756577-wx4s6   1/1     Running   0          54s   10.244.2.9    kind-worker3   <none>           <none>
```

重上述的結果可以觀察到 **10.244.2.11** 以及更以前的(.2~.10) 地址全部都重複了。
到這邊為止，我們透過一個奇怪的操作流程來製造了 **IP** 重複的現象，但是目前還有兩個問題

1. 為什麼這個過程會導致 IP 重複?
2. 這個過程看起來很妙，什麼樣的實際情況會發生?

# 探究問題

從這邊開始，我們就認真地去探討這個問題的本質，一旦理解整個 kubernetes 運作流程後，面對這個問題就會非常有想法知道該從何下手，以及可以很快的縮小問題的發生點

問題開始前，我們要先有一個基本的概念，到底 **IP** 地址是怎麼來的？

實際上 Kubernetes 本身並不會去幫忙分配這些 **IP** 地址，唯二會做就兩件事情
1. Controller 會根據參數設定 Node 上的 NodeCIDR 等數值(本文不談)
2. Kubelet 創建 Pod 時，最後會呼叫起 CNI 來幫忙處理

**Container Network Interface(CNI)** 這邊不談太多，想要瞭解更多可以閱讀我之前關於 CNI 的[介紹文](https://www.hwchiu.com/cni.html)

接下來我們用圖表看一下 CNI 的運作流程(大概流程，實際上底層的呼叫有一點點不一樣)

一開始， **kubelet** 會創造一個 Pod 的沙盒，這時候該 Pod 還沒有任何 IP 地址
![](https://i.imgur.com/sBiUu0E.jpg)

CNI 本身會要負責給予 Pod 對應的網路能力，這邊也包含 IP 的分配，預設情況下 CNI 的設定檔案會放在 /etc/cni/net.d。

**kubelet** 會去讀取 **/etc/cni/net.d** 來判斷當前系統使用的 CNI 是誰
![](https://i.imgur.com/Sgc0odM.jpg)

```bash=
→ docker exec kind-worker3 cat /etc/cni/net.d/10-kindnet.conflist

{
        "cniVersion": "0.3.1",
        "name": "kindnet",
        "plugins": [
        {
                "type": "ptp",
                "ipMasq": false,
                "ipam": {
                        "type": "host-local",
                        "dataDir": "/run/cni-ipam-state",
                        "routes": [
                                {
                                        "dst": "0.0.0.0/0"
                                }
                        ],
                        "ranges": [
                        [
                                {
                                        "subnet": "10.244.2.0/24"
                                }
                        ]
                ]
                }
                ,
                "mtu": 1500

        },
        {
                "type": "portmap",
                "capabilities": {
                        "portMappings": true
                }
        }
        ]
}
```

上面範例是 **kind** 裡面的 CNI 的設定檔案，我們注意 **IPAM** 欄位，他使用的是 **host-local** 這個服務，並且將資料放到 **/run/cni-ipam-state**。


所以 **kubelet** 就會根據找到的結果，叫起 **kind-net** 這套 CNI 來幫忙處理，而 **kind-net** 找上 **host-local** 來幫忙處理 IP 分配問題
![](https://i.imgur.com/zTyuUPX.jpg)


**host-local** 會使用本地檔案 **/run/cni-ipam-state/kindnet** 作為本地資料庫，紀錄哪些 IP 已經使用過。最後根據搜尋從裡面結果挑出一個還沒有被用過的 **IP** 地址，並且把該結果送回給 **kind-net** CNI
![](https://i.imgur.com/daK2V1U.jpg)


```bash=
○ → docker exec kind-worker3 ls /run/cni-ipam-state/kindnet
10.244.2.10
10.244.2.11
10.244.2.12
10.244.2.13
10.244.2.14
10.244.2.15
10.244.2.16
10.244.2.17
10.244.2.18
10.244.2.19
10.244.2.2
10.244.2.20
10.244.2.21
10.244.2.3
10.244.2.4
10.244.2.5
10.244.2.6
10.244.2.7
10.244.2.8
10.244.2.9
last_reserved_ip.0
lock
```

可以看到這邊會記錄當前使用的所有 **CNI** 狀態，這邊要特別注意的是，不是每個 **CNI/IPAM** 都是走 **host-local** 這種方式。 **host-local** 如其名稱一樣，就是每個節點獨立自行處理，而背後就是依賴一個資料夾搭配眾多檔案來記住當前到底分配哪些 **IP 地址**



一切完畢之後，**kind-net** 就會把相關的 **IP** 給設定到 **Pod** 上面。
![](https://i.imgur.com/9qkqNyQ.jpg)


## 再看一次

所以回到前面的模擬環境，現在要理解整個問題流程就非常清楚明瞭了。
一開始，我們部署大量的 Pod 到 kind-worker3 上面， **host-local** 開始針對每個分配的 **IP** 去寫檔案，並且記錄到  **/run/cni-ipam-state/kindnet**。

接者我們停掉 kubelet，讓 CNI 暫時不會被呼叫，然後把維護 **host-local** 的所有狀態都移除了。

到這個階段，其實系統已經出現了狀態不一致的現象， **CNI** 用來維護 **IP** 的狀態資訊被移除，但是使用那些 **IP** 的容器都還活者。

最後，我們透過重新部署 **Pod** 到 **kind-worker3** 上面，這時候 **host-local** 會嘗試重新產生 **IP** 地址，由於 **/run/cni-ipam-state/kindnet** 裡面是空的，所以又會重新開始計算，因此分配出來的 **IP** 就會跟以前重複，產生 IP 衝突問題。

# 真實情境

看到這邊，對於整個問題的發生心理都已經有了個底，但是我們真實情況下會發生這類型的問題嗎?

要滿足這個問題有幾個條件
1. 採用的 CNI 最後會使用 host-local 這種本地資料庫來記錄使用的 Pod IP 地址
2. 系統上 **host-local** 使用的資料夾被刪除，導致裡面的資料全部消失，而 **host-local** 根本不知道

要踩到條件一其實很簡單，預設情抗下 **Flannel** 就是使用這個方式來記錄的
至於條件二，如果系統檔案沒有遇到不如預期的修改的話，通常不會踩到這個點。

但是今天有一個使用情境會完全滿足上述條件，就是透過 **Rancher** 安裝 Kubernetes 並且使用 **Flannel** 作為 CNI。 然後透過 **Restore** 的功能來復原 Kubernetes，這種情況下就會踩到這個問題。


Rancher 再進行 Restore 的過程中，會先關掉 **kubelet**，並且對當節點進行[一番清除](https://github.com/rancher/rke/blob/2c270fa5ab2556e9eec3f2079cb6f2075d6dbfa9/hosts/hosts.go#L67-L82)，譬如
```golang=

const (
	ToCleanEtcdDir          = "/var/lib/etcd/"
	ToCleanSSLDir           = "/etc/kubernetes/"
	ToCleanCNIConf          = "/etc/cni/"
	ToCleanCNIBin           = "/opt/cni/"
	ToCleanCNILib           = "/var/lib/cni/"
	ToCleanCalicoRun        = "/var/run/calico/"
...
)


func (h *Host) CleanUpAll(ctx context.Context, cleanerImage string, prsMap map[string]v3.PrivateRegistry, externalEtcd bool) error {
	log.Infof(ctx, "[hosts] Cleaning up host [%s]", h.Address)
	toCleanPaths := []string{
		path.Join(h.PrefixPath, ToCleanSSLDir),
		ToCleanCNIConf,
		ToCleanCNIBin,
		ToCleanCalicoRun,
		path.Join(h.PrefixPath, ToCleanTempCertPath),
		path.Join(h.PrefixPath, ToCleanCNILib),
	}

	if !externalEtcd {
		toCleanPaths = append(toCleanPaths, path.Join(h.PrefixPath, ToCleanEtcdDir))
	}
	return h.CleanUp(ctx, toCleanPaths, cleanerImage, prsMap)
}
```

該函式就會把 **ToCleanCNILib** 這個路徑給移除掉，而 **ToCleanCNILib** 指向的位置則是  **/var/lib/cni/**。

非常巧的是 [host-local](https://github.com/containernetworking/plugins/blob/ded2f1757770e8e2aa41f65687f8fc876f83048b/plugins/ipam/host-local/backend/disk/backend.go#L31-L39) 預設的路徑也是 **/var/lib/cni**

```golang=
var defaultDataDir = "/var/lib/cni/networks"

// Store is a simple disk-backed store that creates one file per IP
// address in a given directory. The contents of the file are the container ID.
type Store struct {
	*FileLock
	dataDir string
}

```

兩個事情很巧的撞再一起之後，就會產生 IP 衝突的問題了。

# 解決方式

這個問題從兩個人的角度來看，其實都不覺得自己有錯，都很盡忠職守。
但是問題出在，對於 CNI/IPAM 來說，我用來維護整個 IP 狀態的檔案被移除了，可是使用那些 **IP** 的容器卻沒有被移除。

所以一種做法就是把該節點上的所有 Pod 都重啟一次，讓 **CNI** 重新跑過。
或是我們透過 Rancher 安裝 Flannel 的時候，修改設定檔案讓 IPAM(host-local)  使用不同的位置。

畢竟 Rancher (2.5以前)是基於 etcd 的概念去還原與備份整個 Kubernetes Cluster，因此這個過程將節點上的資料都清除是非常合理。

但是 etcd 的資料並不能完整呈現整個 Kubernetes Cluster，畢竟還有一些資料並不存在 etcd 內。
譬如本文章的 CNI，或是其他的 PV/PVC 等，想要做到 k8s 完全備份與還原幾乎是不太可能的，這部分除了技術問題外，我覺得更多是哲學問題，要先定義你想要的備份與還原。


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
