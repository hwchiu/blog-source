---
title: '[Kubernetes] Static Pod 介紹'
keywords: 'Kubernetes,Container, Linux, Pod'
abbrlink: 37539
date: 2020-03-10 07:05:00
tags:
  - Kubernetes
  - Container
  - Linux
description: 本文主要跟大家分享如何透過 Static Pod 的方式來滿足用 Kuberentes 管理 API-Server/Controller/Scheduler 這些 Kubernetes 的基礎元件，其中 Static Pod 更是 Kubeadm 的架設原理，透過這個方式我們也可以更加了解 Kubeadm 的安裝方式
---

# Preface
本文想要跟大家來分享討論一下另外一種部署 Kubernetes Pod 的方式，稱之為 Static Pod，這個部署方式最大的示範情境就是 Kubeadm 的使用。

當部署完 Kubeadm 後，透過 `kubectl -n kube-system get pods`，是不是會看到 `kube-scheduler`, `kube-apiserver` 以及 `kube-controller-manager`.

那..這些核心元件組成了 Kubernetes Control-Plane，但是本身卻又是被 Kubernetes 所管理，那到底是這中間是怎麼運作的?

這個問題就要從 Static Pod 的部署來談起


# Environment
本文觀察環境基於下列版本
1. kubeadm: v1.17.3
2. kubectl: v1.17.3
3. Kubernetes: v1.17.3

如果有使用 Vagrant 的人，可以用下列的檔案建置相關環境

```ruby=
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-18.04"
  config.vm.box_version ='201912.14.0'
  config.vm.hostname = 'k8s-dev'
  config.vm.define vm_name = 'k8s'

  config.vm.provision "shell", privileged: false, inline: <<-SHELL
    set -e -x -u
    export DEBIAN_FRONTEND=noninteractive
    #change the source.list
    sudo apt-get update
    sudo apt-get install -y vim git cmake build-essential tcpdump tig jq socat bash-completion
    # Install Docker
    export DOCKER_VERSION="5:19.03.5~3-0~ubuntu-bionic"
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install -y docker-ce=${DOCKER_VERSION}
    sudo usermod -aG docker $USER
    #Disable swap
    #https://github.com/kubernetes/kubernetes/issues/53533
    sudo swapoff -a && sudo sysctl -w vm.swappiness=0
    sudo sed '/vagrant--vg-swap/d' -i /etc/fstab

    sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee --append /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
    echo 'source <(kubectl completion bash)' >>~/.bashrc
  SHELL

  config.vm.network :private_network, ip: "172.17.8.111"
  config.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--cpus", 2]
      v.customize ["modifyvm", :id, "--memory", 4096]
      v.customize ['modifyvm', :id, '--nicpromisc1', 'allow-all']
  end
end

```



# How Kubeadm Works

為了部署一個 Pod 到 Kubernetes 節點上，其中牽扯了多個元件，從 API Server, Scheduler, Controller 到節點上的 kubelet, Container Runtime。

然而對於 Scheduler/Controller/API Server 這三個核心元件說，到底該怎麼建制以及維護？

1. Native Application, 直接運行三個不同的 Binary 執行檔案
    - 可透過 systemd 來包裝這些應用程式
3. 透過已經包裝好的 Container Image 來執行這三個服務

以 `Kubeadm` 為範例，其先透過 `systemd` 的方式來管理 `kubelet`，確保 `kubelet` 這個 daemon 本身可以被監控，如果有問題會自動重新起動，甚至重新開機後都可以滿足叫起來提供服務。

```bash=
vagrant@k8s-dev:~$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Tue 2020-03-10 04:42:30 UTC; 1h 20min ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 646 (kubelet)
    Tasks: 21 (limit: 4659)
   CGroup: /system.slice/kubelet.service
           └─646 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kub
```



至於上述三個元件本身都有相關的容器映像檔，其實最簡單的方式就是於節點上直接透過 Container Runtime 去運行這三個節點即可。

可以看到安裝完畢 kubeadm 的環境後，系統上都有這些相關的 container image.
```bash=
vagrant@k8s-dev:~$ docker images | grep k8s.gcr
k8s.gcr.io/kube-proxy                v1.17.3             ae853e93800d        3 weeks ago         116MB
k8s.gcr.io/kube-controller-manager   v1.17.3             b0f1517c1f4b        3 weeks ago         161MB
k8s.gcr.io/kube-apiserver            v1.17.3             90d27391b780        3 weeks ago         171MB
k8s.gcr.io/kube-scheduler            v1.17.3             d109c0821a2b        3 weeks ago         94.4MB
k8s.gcr.io/coredns                   1.6.5               70f311871ae1        4 months ago        41.6MB
k8s.gcr.io/etcd                      3.4.3-0             303ce5db0e90        4 months ago        288MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        2 years ago         742kB
```

為了能夠運行起整個 `Kubernetes`, 只要透過(以範例來說) **docker start** 的方式並且給予相關的參數，就能夠將整個 API-Server/Controller/Schduler 運行起來搭建一個 Kubernetes 叢集。

然而實際上你透過下列指令你卻會發現 `kube-system` 中卻有這三個容器的存在
```bash=
vagrant@k8s-dev:~$ kubectl -n kube-system get pods
NAME                              READY   STATUS    RESTARTS   AGE
coredns-6955765f44-gdvrd          1/1     Running   1          15d
coredns-6955765f44-p4wj2          1/1     Running   1          15d
etcd-k8s-dev                      1/1     Running   1          15d
kube-apiserver-k8s-dev            1/1     Running   1          15d
kube-controller-manager-k8s-dev   1/1     Running   1          15d
kube-flannel-ds-amd64-k2w8g       1/1     Running   1          15d
kube-proxy-6nnrt                  1/1     Running   1          15d
kube-scheduler-k8s-dev            1/1     Running   1          15d
```


更特別的是，由下列幾種方式可以推論出基本上沒有使用更上層的管理 **Deployment**, **ReplicaSet**, **DaemonSet**, **StatefulSet**, **ReplicatController**
1. Pod 的命名規則
2. Pod 裡面的 ownerReference
```bash=
vagrant@k8s-dev:~$ kubectl -n kube-system get pod kube-scheduler-k8s-dev -o json | jq '.metadata.ownerReferences'
[
  {
    "apiVersion": "v1",
    "controller": true,
    "kind": "Node",
    "name": "k8s-dev",
    "uid": "b8755102-968b-41ac-a923-0e2cceacaf03"
  }
]
```
3. kubectl -n kube-system get all


對一個沒有任何更高階的 **Controller** 管理的 **Pod** 來說，你如果嘗試將這些 **Pod** 移除，你會發現這些 **Pod** 都會自己重生
```bash=
vagrant@k8s-dev:~$ kubectl -n kube-system get pods -l component=kube-controller-manager
NAME                              READY   STATUS    RESTARTS   AGE
kube-controller-manager-k8s-dev   1/1     Running   1          15d
vagrant@k8s-dev:~$ kubectl -n kube-system delete pod kube-controller-manager-k8s-dev
pod "kube-controller-manager-k8s-dev" deleted
vagrant@k8s-dev:~$ kubectl -n kube-system get pods -l component=kube-controller-manager
NAME                              READY   STATUS    RESTARTS   AGE
kube-controller-manager-k8s-dev   0/1     Pending   0          3s
```

但是如果你自己創立一個 **pod** 並且刪除，就真的完全刪除不會重啟。
```bash=
vagrant@k8s-dev:~$ kubectl get pods
No resources found in default namespace.
vagrant@k8s-dev:~$ kubectl run --generator=run-pod/v1 nginx --image=nginx
pod/nginx created
vagrant@k8s-dev:~$ kubectl get pods
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     ContainerCreating   0          3s
vagrant@k8s-dev:~$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          6s
vagrant@k8s-dev:~$ kubectl get pods nginx -o json | jq '.metadata.ownerReferences'
null
vagrant@k8s-dev:~$ kubectl delete pod nginx
pod "nginx" deleted
vagrant@k8s-dev:~$ kubectl get pods
No resources found in default namespace.
vagrant@k8s-dev:~$
vagrant@k8s-dev:~$
vagrant@k8s-dev:~$ kubectl get pods
No resources found in default namespace.
vagrant@k8s-dev:~$
```

這其中的奧妙就在於 **ownerReferences**，自行創立的 **pod** 是完全空的，但是 **kubeadm** 裡面的 **API Server/Controller/Scheduler** 卻是有資料，並且資料是
```json=
[
  {
    "apiVersion": "v1",
    "controller": true,
    "kind": "Node",
    "name": "k8s-dev",
    "uid": "b8755102-968b-41ac-a923-0e2cceacaf03"
  }
]
```

其實這邊已經透漏出了玄機，這些 **Pod** 是由 **節點Node** 本身去維護的，本身不依賴任何我們到的 workload 型態。 節點取代了過往的 **Kuberentes Controller** 去確保三個核心功能的 Pod 必須活者

而這個用法就是所謂的 Static Pod

# Static Pod

相對於透過 **Kubernetess** 控制平面來管理這些 **Pod**, **Static Pod** 有一些特性
1. 沒有 Schedule 的概念，就是固定於該節點運行
2. 由 Kubelet 去進行監控並且管理，一旦該 Pod 結束則會重新啟動
3. Kubelet 本身會 Mirror 該 Pod 的資訊，所以才可以透過 kubectl 等相關資訊去看到

其中如果(3)的部分可以參閱 [kubelet 原始碼](https://github.com/kubernetes/kubernetes/blob/v1.17.3/pkg/kubelet/kubelet.go#L1638)
```golang=
	// Create Mirror Pod for Static Pod if it doesn't already exist
	if kubetypes.IsStaticPod(pod) {
		podFullName := kubecontainer.GetPodFullName(pod)
		deleted := false
		if mirrorPod != nil {
			if mirrorPod.DeletionTimestamp != nil || !kl.podManager.IsMirrorPodOf(mirrorPod, pod) {
				// The mirror pod is semantically different from the static pod. Remove
				// it. The mirror pod will get recreated later.
				klog.Infof("Trying to delete pod %s %v", podFullName, mirrorPod.ObjectMeta.UID)
				var err error
				deleted, err = kl.podManager.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
				if deleted {
					klog.Warningf("Deleted mirror pod %q because it is outdated", format.Pod(mirrorPod))
				} else if err != nil {
					klog.Errorf("Failed deleting mirror pod %q: %v", format.Pod(mirrorPod), err)
				}
			}
		}
		if mirrorPod == nil || deleted {
			node, err := kl.GetNode()
			if err != nil || node.DeletionTimestamp != nil {
				klog.V(4).Infof("No need to create a mirror pod, since node %q has been removed from the cluster", kl.nodeName)
			} else {
				klog.V(4).Infof("Creating a mirror pod for static pod %q", format.Pod(pod))
				if err := kl.podManager.CreateMirrorPod(pod); err != nil {
					klog.Errorf("Failed creating a mirror pod for %q: %v", format.Pod(pod), err)
				}
			}
		}
	}
```

對於 Kubelet 來說，其本身有一個設定叫做 `staticPodPath`, 這是一個資料夾，只要放到該資料夾下的檔案都會被 **kubelet** 用來創立 **static pod**.

至於 **Kubelet** 創造 **Pod** 的方式其實還是遵循 **Kubernetes** 的走法，並非直接使用 **docker start**(舉例) 來創立。這部分是為了讓所有的 **Container** 操作全部都經由 **Container Runtime Interface** 來管理，進而提供更好的相容性。


# Hands-on

接下來我們就動手觀察一下相關的參數，首先根據剛剛 **systemd** 的提示，我們知道用來控管 **kubelet** 的啟動檔案於 **/etc/systemd/system/kubelet.service.d/10-kubeadm.conf**

```bash=
vagrant@k8s-dev:~$ sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

裡面要注意的是 **KUBELET_CONFIG_ARGS** 這個變數，裡面使用了 **--config=/var/lib/kubelet/config.yaml** 來指名 **kubelet** 的參數檔案

因此接下來看一下其內容
```bash=
vagrant@k8s-dev:~$ sudo cat  /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

其中吸引我們注意的是  **staticPodPath** 這個參數，接下來看一下該資料夾的位置

```bash=
vagrant@k8s-dev:~$ sudo ls /etc/kubernetes/manifests
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

可以看到裡面放了四個 **yaml** 檔案(包含 etcd)，如果打開這些 **yaml** 就會是我們所熟悉的 **Kubernetes** 格式了。

如果這時候隨便放入一個 **yaml** 到該資料夾中，會發生什麼事情?

```bash=

cat <<EOF | sudo tee /etc/kubernetes/manifests/static-debug.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-debug
spec:
  containers:
    - name: hwchiu
      image: hwchiu/netutils
EOF

vagrant@k8s-dev:~$ kubectl get  pods
NAME                   READY   STATUS    RESTARTS   AGE
static-debug-k8s-dev   1/1     Running   0          27s
vagrant@k8s-dev:~$ kubectl delete pod static-debug-k8s-dev
pod "static-debug-k8s-dev" deleted
vagrant@k8s-dev:~$ kubectl get  pods
NAME                   READY   STATUS    RESTARTS   AGE
static-debug-k8s-dev   0/1     Pending   0          1s
vagrant@k8s-dev:~$ kubectl get  pods
NAME                   READY   STATUS    RESTARTS   AGE
static-debug-k8s-dev   1/1     Running   0          3s
```
這邊可以看到馬上就會產生對應的 **Pod** 並且也獲得了自動重啓的能力。


最後！ 我們透過 **docker  ps** 觀察一下
```bash=
vagrant@k8s-dev:~$ docker ps | grep kube-controller
5f16cb76460f        b0f1517c1f4b           "kube-controller-man…"   2 hours ago         Up 2 hours                              k8s_kube-controller-manager_kube-controller-manager-k8s-dev_kube-system_25245994bd78f09602b6f5c3e5d2246c_1
a94f104316af        k8s.gcr.io/pause:3.1   "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_kube-controller-manager-k8s-dev_kube-system_25245994bd78f09602b6f5c3e5d2246c_1vagrant@k8s-dev:~$ docker ps | grep kube-controller
5f16cb76460f        b0f1517c1f4b           "kube-controller-man…"   2 hours ago         Up 2 hours                              k8s_kube-controller-manager_kube-controller-manager-k8s-dev_kube-system_25245994bd78f09602b6f5c3e5d2246c_1
a94f104316af        k8s.gcr.io/pause:3.1   "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_kube-controller-manager-k8s-dev_kube-system_25245994bd78f09602b6f5c3e5d2246c_1
```
可以看到針對 **kube-controller-manager**  這個 **Pod** 來說，其實背後也是有 **Pause Container** 作為整個 **Pod** 的沙盒，這也證明了這些 **Static Pod** 的創建也是基於 **CRI** 的標準所創立的，並非是直接透過 **docker command** 來創立。
  - CRI 內的基本單位都是 Pod, 而非 Container, 有興趣的可以參考他們的 gRPC 介面


另外，其實 **kubeadm** 的安裝過程就已經透露出相關資訊
```bash=
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0310 06:57:46.902862    2798 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0310 06:57:46.903651    2798 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifes
ts". This can take up to 4m0s
```



# Summary
這次我們探討了關於 **Static Pod** 的概念，雖然實際部署上比較少這樣使用，但是透過這次的探討我們可以更了解 **Kubeadm** 是如何安裝 **Kubernetes** 環境，
並且也學習到了一些 **Kubernetes** 本身的特性。

# 課程分享
最後，我目前於 Hiskio 上面有開設一門 Kubernetes 入門篇的課程，裡面會探討運算/網路/儲存三個最重要的平台資源，此外對於 CRI/CNI/CSI 也都有簡單的介紹，主要會基於 **Kubernetes** 本身的設計原理及各資源的用法與情境去介紹。
如果本身已經很熟練的使用 Kubernetes 於環境中就不太適合這門課程，主要是給想要踏入到 Kubernetes 世界中的朋友，有興趣的幫忙捧場或推廣
組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=LG28Q5G

單堂(CI/CD)
https://hiskio.com/courses/385?promo_code=13K49YE&p=blog1

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE


# Reference
- https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/
- https://github.com/kubernetes/kubernetes/blob/v1.17.3/pkg/kubelet/kubelet.go#L1637-L1638

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
