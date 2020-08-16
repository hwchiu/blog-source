---
title: IPvS 學習手冊(二)
tags:
  - IPVS
  - Network
  - Linux
keywords: 'Kubernetes,Network,Linux,Ubuntu, IPVS'
description: >-
  本文作為 IPVS 系列文第二篇，主要是跟大家介紹 IPVS 與 Kubernetes 的互動，包含如何設定以及 IPVS 如何實踐 Kubernetes
  Service 的功能
abbrlink: 58431
date: 2020-03-21 06:38:55
---



# Preface
本篇文章作為系列文章的第二篇，該系列文希望能夠從概念到實作，從簡單到複雜來探討 IPVS (IP Virtual Server) 的概念，目前規劃的主題包含：
- [IPVS 的基本使用與概念](https://www.hwchiu.com/ipvs-1.html)
- [IPVS 與 Kubernetes 的整合](https://www.hwchiu.com/ipvs-2.html)
- [IPVS 除錯方式與基本 Kernel Module 概念](https://www.hwchiu.com/ipvs-3.html)
- [IPVS Kernel 架構實現](https://www.hwchiu.com/ipvs-4.html)


本文主要是從 Kubernetes 出發，介紹如何在 Kubernetes 內使用 IPVS 而非原生的 IPTables，並且探討下 **Kubernetes Service** 是如何透過 **IPVS** 實踐的。

本文中使用的所有檔案都可以於 [network-study:ipvs](https://github.com/hwchiu/network-study/tree/master/ipvs) 這個 repo找到，裡面包含了
1. 建設環境用的 Vagrant 檔案
2. 部署 Kubernetes Service 會用到的相關 yaml 檔案

# Kubernetes
Kube-Proxy 是 Kubernetes 用來控制 Service 轉發過程的一個元件，基本上每個節點上都要部署該元件。預設情況下， **kube-proxy** 會使用 **iptables** 作為 **Kubernetes Service** 的底層實現方式，而我們可以透過參數變化的方式要求其使用 **IPVS**。

為了快速架設環境，我準備了一個基於 **kubeadm** 的安裝環境，內容如下
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
    sudo apt-get install -y vim git cmake build-essential tcpdump tig jq socat bash-completion ipvsadm
    # Install Docker
    export DOCKER_VERSION="5:19.03.5~3-0~ubuntu-bionic"
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install -y docker-ce=${DOCKER_VERSION}
    sudo usermod -aG docker $USER
    #Install module
    sudo modprobe -- ip_vs
    sudo modprobe -- ip_vs_rr
    sudo modprobe -- ip_vs_wrr
    sudo modprobe -- ip_vs_sh
    sudo modprobe -- nf_conntrack_ipv4
    #Disable swap
    #https://github.com/kubernetes/kubernetes/issues/53533
    sudo swapoff -a && sudo sysctl -w vm.swappiness=0
    sudo sed '/vagrant--vg-swap/d' -i /etc/fstab
    git clone https://github.com/hwchiu/network-study.git
    sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee --append /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo kubeadm init --config network-study/ipvs/kubeconfig.yaml
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
    kubectl taint node k8s-dev node-role.kubernetes.io/master:NoSchedule-
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

裡面有幾個比較值得注意的點
1. apt-get 順便安裝 ipvsadm 與 ipset
2. 透過 `modprobe` 事先安裝相關的 **kernel module**, 包含了
    - ip_vs
    - ip_vs_rr
    - ip_vs_wrr
    - ip_vs_sh
    - nf_conntrack_ipv4
    > 其中有幾個主要都是 **Load Balancing** 不同演算法實踐的 Module，之後探討底層實作的時候再來細看這些的差異。
3. kubeadm init 的時候透過 `--config` 來預載設定檔案，其內容如下：
```yaml=
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"
```

透過 `KubeProxtConfiguration` 的方式讓 `kubeadm` 產生對應的 **configmap** 給 **kube-proxy** 使用。

```bash=
vagrant@k8s-dev:~$ kubectl -n kube-system get configmaps kube-proxy
NAME         DATA   AGE
kube-proxy   2      105m
```

而 **ClusterConfiguration** 則是因為底下使用 **Flannel CNI** 所以需要設定的 **POD CIDR** 的參數

透過 **vagrant up** 將環境建置起來後，可以透過下列指令觀察最原始的 **Kubernetes** 設定。

```bash=
vagrant@k8s-dev:~$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 10.0.2.15:6443               Masq    1      3          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
  -> 10.244.0.3:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.0.2:9153              Masq    1      0          0
  -> 10.244.0.3:9153              Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
  -> 10.244.0.3:53                Masq    1      0          0
```

可以觀察到，預設情況下，系統中有四個 **IPVS Service**
1. **10.96.0.1:443** (TCP)
2. **10.96.0.10:53** (TCP)
3. **10.96.0.10:9153** (TCP)
4. **10.96.0.10:53** (UDP)

第一個則是 API Server 的服務，沒有太多的重點，主要是觀察後面三個服務
這三個服務都是對應 `CoreDNS` 的服務，可以透過下列指令觀察

```bash=
vagrant@k8s-dev:~$ kubectl -n kube-system get svc,ep
NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   126m

NAME                                ENDPOINTS                                               AGE
endpoints/kube-controller-manager   <none>                                                  126m
endpoints/kube-dns                  10.244.0.2:53,10.244.0.3:53,10.244.0.2:53 + 3 more...   126m
endpoints/kube-scheduler            <none>                                                  126m
```

所以可以看到，對於 **IPVS**來說，每一個 **Kubernetes Service** 可以產生多個 **IPVS Service**，主要看該 **Service** 要提供多少服務（協定+連接埠）。

# Service
接下來我們嘗試部署 ClusterIP 以及 NodePort 這兩個不同類型的服務，看看 **IPVS** 本身會有什麼改變。

## ClusterIP/NodePort

```bash=
vagrant@k8s-dev:~$ kubectl apply -f network-study/ipvs/service.yml -f network-study/ipvs/hello.yml

vagrant@k8s-dev:~$ kubectl get svc,ep
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/cluster-demo    ClusterIP   10.97.35.96     <none>        80/TCP         25s
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        18m
service/nodeport-demo   NodePort    10.100.61.228   <none>        80:31543/TCP   25s

NAME                      ENDPOINTS                                         AGE
endpoints/cluster-demo    10.244.0.4:8080,10.244.0.5:8080,10.244.0.6:8080   25s
endpoints/kubernetes      10.0.2.15:6443                                    18m
endpoints/nodeport-demo   10.244.0.4:8080,10.244.0.5:8080,10.244.0.6:8080   25s
```

先確認一下
1. ClusterIP 使用的是 **10.97.35.96**
2. NodePort 使用的是 **10.100.61.228** 同時使用的 Port是 **31543**。

由於輸出過長，我們就忽略 **Real Servers** 的部分，專心看 **IPVS Service**。

```bash=
vagrant@k8s-dev:~$ sudo ipvsadm -Ln | grep rr
TCP  10.96.0.1:443 rr
TCP  10.96.0.10:53 rr
TCP  10.96.0.10:9153 rr
TCP  10.97.35.96:80 rr
TCP  10.100.61.228:80 rr
TCP  127.0.0.1:31543 rr
TCP  172.17.8.111:31543 rr
TCP  172.18.0.1:31543 rr
TCP  10.0.2.15:31543 rr
TCP  10.244.0.0:31543 rr
TCP  10.244.0.1:31543 rr
UDP  10.96.0.10:53 rr
```

首先 **ClusterIP** 的部分非常簡單，就 `TCP  10.97.35.96:80 rr` 一個規則而已，但是對於 **NodePort** 來說，這邊則是要針對系統上全部的 **IP** 都去設定，所以會看到總共有六個 **IPVS Service**，分別對應系統上六個**IP**，且都指向 **31543** 這個連接埠。

```
TCP  127.0.0.1:31543 rr
TCP  172.17.8.111:31543 rr
TCP  172.18.0.1:31543 rr
TCP  10.0.2.15:31543 rr
TCP  10.244.0.0:31543 rr
TCP  10.244.0.1:31543 rr
```

藉由上述的觀察，我們可以知道 **IPVS** 目前創造的規則如同下方所述，首先讓我們假設該 Serivce 有開放 `n` 個連接方式(L3+L4)：

1. 對於 ClusterIP 會創造 **n** 個 **IPVS Service**
2. 對於 NodePort 來說, 對於系統上每一個網卡，都會創造 **n** 個 **IPVS Service**，所以假如系統中有五個對外 IP, 那就會有 `5*n` 個 **IPVS Service**。

## IPSET

除了 **IPVS Service** 以及 **IPVS Real Servers** 的組合外，kube-proxy(IPVS) 本身也會透過 **IPSET** 來輔佐整個網路連線的處理，舉例來說：
1. 防火牆
2. SNAT (Masquerade)

這兩個功能還是要依賴 **IPtables** 來幫忙完成，但是這邊為了讓 **IPtables** 的規則盡量得少，不想要每一個 **IP** 就一條規則，進而提升整體規則的匹配效能，於是採用了 **IPSET** 的方式來幫忙處理。

用一個最快速的方式來講就是將一堆 **IP:PORT** 透過不同的方式放到一個 **SET** 裡面，而 **IPTABLES** 本身就針對這個 **SET** 去比較。 直接以下列範例來看

```bash=
vagrant@k8s-dev:~$ sudo ipset list KUBE-NODE-PORT-TCP
Name: KUBE-NODE-PORT-TCP
Type: bitmap:port
Revision: 3
Header: range 0-65535
Size in memory: 8268
References: 1
Number of entries: 1
Members:
31543

vagrant@k8s-dev:~$ sudo ipset list KUBE-CLUSTER-IP
Name: KUBE-CLUSTER-IP
Type: hash:ip,port
Revision: 5
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 472
References: 2
Number of entries: 6
Members:
10.96.0.10,udp:53
10.96.0.10,tcp:53
10.96.0.10,tcp:9153
10.97.35.96,tcp:80
10.100.61.228,tcp:80
10.96.0.1,tcp:443
```

上面可以看到兩組 **IPSET**，其中第一組是針對 **NODE PORT** 去使用，其型態為 **bitmap:port**，這部分只針對 **Port** 去比對，所以裡面可以看到 `31543` 而已

至於第二組則是針對所有的 **ClusterIP** 去使用，他的型態則是 **hash:ip,port**，所以每個資料都是 **IP:Port** 的規則，可以看到我們之前用到的全部 **ClusterIP:Port** 都在裡面。

有了這兩組 **IPSET** 後，我們稍微看一下 **IPTABLES** 會怎麼使用 **IPSET** KUBE-NODE-PORT-TCP 來減少需要的規則數量。

### NODE-PORT
```bash=
vagrant@k8s-dev:~$ sudo iptables-save  | grep KUBE-NODE
-A KUBE-NODE-PORT -p tcp -m comment --comment "Kubernetes nodeport TCP port for masquerade purpose" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT
```

先觀察的第一個規則是 `KUBE-SERVICES`，本身使用 **--dst-type LOCAL** 來判斷封包是不是針對 `LOCAL` (本地網卡)是的話就跳到`KUBE-NODE-PORT` 去二次處理。
`KUBE-NODE-PORT` 裡面透過 `-m set --match-set KUBE-NODE-PORT-TCP dst` 來判斷封包的目標 Port 有在 **KUBE-NODE-PORT-TCP** 這個 **ipset** 裡面，就去弄 **KUBE-MARK-MASQ** 相關的動作。

因為最外層已經透過 **dst-type LOCAL** 來判斷是不是送往本地介面，所以這邊的 **IPSET** 只需要處理 **Port** 即可，就不用管 **IP**。

更多關於 **IPSET** 的使用意思以及使用情境可以參考[官方文件](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs#when-ipvs-falls-back-to-iptables)

## Dummy Interface

最後最後，我們來探討一個有趣的事情，這些 **ClusterIP** 本身都是一個不存在的**IP**, 那我們到底是如何成功地讓封包被 **IPVS** 接手處理的？

```bash
vagrant@k8s-dev:~$ kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
cluster-demo    ClusterIP   10.97.35.96     <none>        80/TCP         63m
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        81m
nodeport-demo   NodePort    10.100.61.228   <none>        80:31543/TCP   63m

vagrant@k8s-dev:~$ curl 10.97.35.96
...
...
```

譬如上述，我們可以在節點內直接用 curl 的方式透過 **clusterIP** 去存取服務，可是之前在探討 **IPVS** 的時候，那時候你的 **Service IP(VIP)** 必須要真實存在才可以處理。
而這些 **ClusterIP** 本身又不在系統之中，那到底怎麼辦？

為了解決這個問題，**kube-proxy**於系統內偷偷創造了一個 **dummy interface kube-ipvs0**，並且把所有的 **clusterIP** 都設定到該網卡上面了，可以用下面的指令觀察到：

```bash=
vagrant@k8s-dev:~$ ip link show type dummy
5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT group default
    link/ether 62:c9:fc:74:c4:f8 brd ff:ff:ff:ff:ff:ff
vagrant@k8s-dev:~$ ip addr show dev kube-ipvs0
5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 62:c9:fc:74:c4:f8 brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.10/32 brd 10.96.0.10 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.1/32 brd 10.96.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.97.35.96/32 brd 10.97.35.96 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.100.61.228/32 brd 10.100.61.228 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```

上述的資料有一個要注意的就是，其狀態 `DOWN`, 這意味該 **Interface** 本身不是一個可運作的狀態，單純只是一個沒有被叫起來運作的 **Interface**。

實際上 **dummy interface** 本身跟 **IPVS** 的協同合作只有一個目的，讓封包可以往 Kernel 送，只要封包可以順利送進去，接下來就可以被 **IPVS** 給接手處理

**這部分的原理要等到下篇文章從 kernel + netfilter 看起才比較好說明原理**

這邊直接快速用一個實驗來驗證上面推論

### Experiment

1. 我們把該 Interface kube-ipvs0 叫起來
2. 隨便捏造一個假的 IP, 並且設定一條靜態路由指向 kube-ipvs0
3. 手動用 ipvsadm 根據上面假的 IP 去創造一個新的 **IPVS Service**
4. 手動將我們的 **pod IP** 加入到上述創造的 **IPVS Service**
5. 透過 Curl 去連接我們創造的假 IP

假設我們捏造一個 `1.2.3.4/32` 的 **IP** 地址，然後後端的 **POD IP:PORT** 是 `10.244.0.4:8080,10.244.0.5:8080,10.244.0.6:8080`。

```bash=
sudo ifconfig kube-ipvs0 up
sudo ip route add 1.2.3.4/32 table local dev kube-ipvs0
sudo ipvsadm -A -t 1.2.3.4:80
sudo ipvsadm -a -t 1.2.3.4:80 -r 10.244.0.4:8080 -m
sudo ipvsadm -a -t 1.2.3.4:80 -r 10.244.0.5:8080 -m
sudo ipvsadm -a -t 1.2.3.4:80 -r 10.244.0.6:8080 -m
curl 1.2.3.4
```

執行完 curl 就會順利的存取到後面伺服器的網頁內容，但是要注意的是
**這邊因為 IPVS 會根據 SyncPeriod 的設定定期去更新規則，所以上述創造的規則放一段時間就會被刪除** 

根據這個實驗可以驗證我們的猜想，其實 **kube-ipvs0** 這個 **interface** 本身根本不需要有任何的 **IP Address**，其目的只是一開始產生的 **IP address** 能夠產生一個對應的 **Route Entry**，把封包往系統內送，當封包走到系統內後，便會與 **Netfilter** 交互作用將封包轉接給 **IPVS** 的底層實作去處理，這時候就會根據 **IPVS** 的 **Service** 來決定是否有匹配的資料並且將其轉發到後端伺服器。


下篇文章就會來開始探討到底 **IPVS** 與 **IPTables** 的差異在哪裏，並且嘗試解釋上面的推論過程其背後的實作原理

# 課程分享

最後，我目前於 Hiskio 上面有開設一門 Kubernetes 入門篇的課程，裡面會探討運算/網路/儲存三個最重要的平台資源，此外對於 CRI/CNI/CSI 也都有簡單的介紹，主要會基於 **Kubernetes** 本身的設計原理及各資源的用法與情境去介紹。
如果本身已經很熟練的使用 Kubernetes 於環境中就不太適合這門課程，主要是給想要踏入到 Kubernetes 世界中的朋友，有興趣的幫忙捧場或推廣
組合包
https://hiskio.com/packages/D7RZGWrNK

單堂(CI/CD)
https://hiskio.com/courses/385?promo_code=13K49YE&p=blog1

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE


# Reference

- https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs
- https://godoc.org/k8s.io/kube-proxy/config/v1alpha1
