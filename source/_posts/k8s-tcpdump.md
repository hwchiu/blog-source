---
title: Kubernetes 之封包去哪兒
keywords: 'Kubernetes,Network,Linux,Ubuntu'
tags:
  - Kubernetes
  - Network
  - Linux
description: 談談 Kubernetes 的世界中如果想要用 tcpdump 抓封包可以怎麼做
abbrlink: 36837
date: 2021-03-05 21:58:52
---

# 前言

今天這篇文章想要跟大家分享一下，平常跟 Kubernetes 搏鬥的日常生活中，如果想要找到相關應用程式的封包可以有哪些方法。

沒有 Kubernetes 的架構中，只要環境單純，大家都會想到使用 tcpdump 或是 wireshark 等工具來幫忙抓取封包，這想法非常直覺且大部分情況下都可以使用。

註: 今天只想要探討 tcpdump 這工具的使用與情境，其他會讓封包消失於 kernel 中的除錯於之後的文章再來探討。


但是到了 Kubernetes 的世界中，這個問題變得棘手，造成棘手的原因我認為有兩個
1. 服務容器化了，如果想要在容器內直接抓取封包變得非常麻煩，因為很多時候該 container image 根本不是自己維護的，內部甚至連 sh 相關指令都沒有 (Traefik, fluent-bit 我就是說你們...)
2. Kubernetes 底下的網路都是透過 CNI 來處理跨節點之間的網路處理，不同的 CNI 的作法完全不同，要如何於每個 Kubernetes 節點上精準地去抓取封包是一個不簡單的問題。

實際上 tcpdump 可以直接針對所有的網卡去抓取封包，但是這種情況下會抓到數量非常誇張的封包，後續必須要用精準的方式去過濾封包，才能夠針對你的需求找到最後的封包。
更重要的是，這個方法不是萬用的，因為如果你的底層 overlay network 使用 tunnling 這種封裝封包的方式來傳輸的話，你第一層看到的封包內容根本不是真正 Pod 的資訊，你簡單的過濾功能根本沒有辦法找到底層的細節。

種種問題造成 Kubernetes 內抓取容器的封包變得麻煩，這邊來跟大家分享從兩個不同的層次來抓取封包
1. 從容器本身抓取封包
2. 從節點精準抓取封包

註
> 本篇文章假設讀者都知道與聽過 tcpdump 這個工具的使用，不會針對參數進行講解，主要會著重於抓取封包的思路。
>
> 使用 SR-IOV, RDMA, DPDK 等特別的網路裝置則要有別的工具與方式，本篇文章的東西大部分都不太適用，有機會可以在探討這些機制下的封包世界。


# 容器抓取封包

想要於容器中抓取封包最簡單的方式就是於容器內運行 tcpdump 這個指令即可，為了完成這個動作，我們有兩個思路可以進行

1. 透過 kubectl exec 的方式進入到容器內直接執行 tcpdump 指令
2. 由於 Kubernetes Pod 內可以運行多個 container，而這些 container 都會與 pause container 共享相同的 network namespace，所以其實可以運行一個 sidecar container，於該 container 上運行 tcpdump 即可。
3. 於節點上運行一個全新的 Container，該容器與目標應用程式的 Container 共用相同的 network namespace，接者於裏面運行 tcpdump 即可。

上述三個概念分別對應於下圖的 PodA, PodB 以及 PodC。

![](https://i.imgur.com/tGt1bfk.png)


第一點聽起來而且但是事物上不太方便，因為現在的 container 基於某種潮流，非常喜歡瘦身，特別喜歡從 scratch 開始搭建環境，基本上除錯時需要的指令沒有半個內建。
很多時候想要自行安裝該指令，卻發現 apk, apt-get 等指令都沒有，這時候可以考慮使用 ksniff 這套工具來幫忙輔助

ksniff 這個工具可以單獨使用，也可以透過 krew 安裝到 kubectl 的生態系中，譬如
```
$ kubectl krew install sniff
$ kubectl sniff
Usage:
  sniff pod [-n namespace] [-c container] [-f filter] [-o output-file] [-l local-tcpdump-path] [-r remote-tcpdump-path] [flags]

Examples:
kubectl sniff hello-minikube-7c77b68cff-qbvsd -c hello-minikube

Flags:
  -c, --container string             container (optional)
  -x, --context string               kubectl context to work on (optional)
  -f, --filter string                tcpdump filter (optional)
  -h, --help                         help for sniff
      --image string                 the privileged container image (optional)
  -i, --interface string             pod interface to packet capture (optional) (default "any")
  -l, --local-tcpdump-path string    local static tcpdump binary path (optional)
  -n, --namespace string             namespace (optional) (default "default")
  -o, --output-file string           output file path, tcpdump output will be redirect to this file instead of wireshark (optional) ('-' stdout)
  -p, --privileged                   if specified, ksniff will deploy another pod that have privileges to attach target pod network namespace
  -r, --remote-tcpdump-path string   remote static tcpdump binary path (optional) (default "/tmp/static-tcpdump")
  -v, --verbose                      if specified, ksniff output will include debug information (optional)

```

sniff(ksniff) 這個工具的思路很簡單，大部分的容器有這個問題就是因為沒有 tcpdump 這個指令，因此 sniff 會幫你上傳 tcpdump 的 binary 到目標容器中，然後於該容器中去運行 tcpdump 這個指令。
舉例來說

```bash=
→ kubectl sniff debug-pod-5cd7fdbc68-xgjzq
INFO[0000] using tcpdump path at: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump'
INFO[0000] no container specified, taking first container we found in pod.
INFO[0000] selected container: 'debug-pod'
INFO[0000] sniffing method: upload static tcpdump
INFO[0000] sniffing on pod: 'debug-pod-5cd7fdbc68-xgjzq' [namespace: 'default', container: 'debug-pod', filter: '', interface: 'any']
INFO[0000] uploading static tcpdump binary from: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump' to: '/tmp/static-tcpdump'
INFO[0000] uploading file: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump' to '/tmp/static-tcpdump' on container: 'debug-pod'
INFO[0000] executing command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' on container: 'debug-pod', pod: 'debug-pod-5cd7fdbc68-xgjzq', namespace: 'default'
INFO[0000] command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' executing successfully exitCode: '0', stdErr :''
INFO[0000] file found: '-rwxr-xr-x 1 root root 2696368 Jan  1  1970 /tmp/static-tcpdump
'
INFO[0000] file was already found on remote pod
INFO[0000] tcpdump uploaded successfully
INFO[0000] spawning wireshark!
INFO[0000] starting sniffer cleanup
INFO[0000] sniffer cleanup completed successfully
Error: exec: "wireshark": executable file not found in $PATH
```

上述可以觀察到幾個重點
1. uploading static tcpdump binary from:  ....
2. executing command ...

預設情況下， sniff(ksniff) 會希望本地環境有安裝 wireshark 這個工具，能夠幫忙將截取的封包直接導入到 wireshark 中去解讀。如果環境中沒有 GUI 視窗的話，請改安裝 tshark 這個工具，並且將輸入導向 tshark

```
→ kubectl sniff debug-pod-5cd7fdbc68-xgjzq -o - | sudo tshark -r -
Running as user "root" and group "root". This could be dangerous.
INFO[0000] using tcpdump path at: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump'
INFO[0000] no container specified, taking first container we found in pod.
INFO[0000] selected container: 'debug-pod'
INFO[0000] sniffing method: upload static tcpdump
INFO[0000] sniffing on pod: 'debug-pod-5cd7fdbc68-xgjzq' [namespace: 'default', container: 'debug-pod', filter: '', interface: 'any']
INFO[0000] uploading static tcpdump binary from: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump' to: '/tmp/static-tcpdump'
INFO[0000] uploading file: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump' to '/tmp/static-tcpdump' on container: 'debug-pod'
INFO[0000] executing command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' on container: 'debug-pod', pod: 'debug-pod-5cd7fdbc68-xgjzq', namespace: 'default'
INFO[0000] command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' executing successfully exitCode: '0', stdErr :''
INFO[0000] file found: '-rwxr-xr-x 1 root root 2696368 Jan  1  1970 /tmp/static-tcpdump
'
INFO[0000] file was already found on remote pod
INFO[0000] tcpdump uploaded successfully
INFO[0000] output file option specified, storing output in: '-'
INFO[0000] start sniffing on remote container
INFO[0000] executing command: '[/tmp/static-tcpdump -i any -U -w - ]' on container: 'debug-pod', pod: 'debug-pod-5cd7fdbc68-xgjzq', namespace: 'default'
    1   0.000000  10.244.0.11 → 8.8.8.8      ICMP 100 Echo (ping) request  id=0x0027, seq=332/19457, ttl=64
    2   0.013479      8.8.8.8 → 10.244.0.11  ICMP 100 Echo (ping) reply    id=0x0027, seq=332/19457, ttl=61 (request in 1)
    3   1.022986  10.244.0.11 → 8.8.8.8      ICMP 100 Echo (ping) request  id=0x0027, seq=333/19713, ttl=64
    4   1.040225      8.8.8.8 → 10.244.0.11  ICMP 100 Echo (ping) reply    id=0x0027, seq=333/19713, ttl=61 (request in 3)
    5   2.031062  10.244.0.11 → 8.8.8.8      ICMP 100 Echo (ping) request  id=0x0027, seq=334/19969, ttl=64
    6   2.043612      8.8.8.8 → 10.244.0.11  ICMP 100 Echo (ping) reply    id=0x0027, seq=334/19969, ttl=61 (request in 5)
```

透過 -o 的參數將輸出導向到 tshark 後可以開始觀測封包了，範例中可以看到有 ICMP (ping) 的來回封包，基本上這樣已經幫助我們解決了一點小問題。

這邊要特別注意一下，基於安全考量，很多容器都開始使用 non-root 的方式去運行，這種情況下 sniff 會遇到問題，以下為範例

``` bash=
→ kubectl sniff debug-pod-6ff57fbd8-tnfzh -o - | sudo tshark -r -                                                                                                  [5/1820]
Running as user "root" and group "root". This could be dangerous.
INFO[0000] using tcpdump path at: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump'
INFO[0000] no container specified, taking first container we found in pod.
INFO[0000] selected container: 'debug-pod'
INFO[0000] sniffing method: upload static tcpdump
INFO[0000] sniffing on pod: 'debug-pod-6ff57fbd8-tnfzh' [namespace: 'default', container: 'debug-pod', filter: '', interface: 'any']
INFO[0000] uploading static tcpdump binary from: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump' to: '/tmp/static-tcpdump'
INFO[0000] uploading file: '/home/vagrant/.krew/store/sniff/v1.5.0/static-tcpdump' to '/tmp/static-tcpdump' on container: 'debug-pod'
INFO[0000] executing command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' on container: 'debug-pod', pod: 'debug-pod-6ff57fbd8-tnfzh', namespace: 'default'
INFO[0000] command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' executing successfully exitCode: '2', stdErr :'ls: cannot access '/tmp/static-tcpdump': No such file or directory
'
INFO[0000] file not found on: '/tmp/static-tcpdump', starting to upload
INFO[0000] verifying file uploaded successfully
INFO[0000] executing command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' on container: 'debug-pod', pod: 'debug-pod-6ff57fbd8-tnfzh', namespace: 'default'
INFO[0000] command: '[/bin/sh -c ls -alt /tmp/static-tcpdump]' executing successfully exitCode: '0', stdErr :''
INFO[0000] file found: '-rwxr-xr-x 1 123 root 2696368 Jan  1  1970 /tmp/static-tcpdump
'
INFO[0000] file uploaded successfully
INFO[0000] tcpdump uploaded successfully
INFO[0000] output file option specified, storing output in: '-'
INFO[0000] start sniffing on remote container
INFO[0000] executing command: '[/tmp/static-tcpdump -i any -U -w - ]' on container: 'debug-pod', pod: 'debug-pod-6ff57fbd8-tnfzh', namespace: 'default'
INFO[0000] command: '[/tmp/static-tcpdump -i any -U -w - ]' executing successfully exitCode: '1', stdErr :'static-tcpdump: any: You don't have permission to capture on that device
(socket: Operation not permitted)
'
```

可以看到最底下直接噴出了 `Operation not permitted`，理由是 `You don't have permission to capture on that device`。 這問題是因為你不是 root 同時`capabilities` 沒有設定好。
為了解決這個問題， sniff 提供了 -p 這個參數，該參數提供了[Non-Privileged](https://github.com/eldadru/ksniff#non-privileged-and-scratch-pods) 的模式處理，不過目前版本會有一個 docker socket 的位置的 bug，要等到 1.5.1 版本釋出才能夠透過參數設定，詳細錯誤訊息可以到 [Github issue](https://github.com/eldadru/ksniff/issues/82) 觀看

第二個方式透過 sidecar 的方式則沒有什麼好敘述的，就 YAML 中多掛一個包含 tcpdump 的容器即可，記得權限要設定好，但是這個方法比較不方便針對已經運行的 Pod，畢竟需要重新部署整個 Pod，可能環境就被破壞了。

最後一個方式也是簡單，假設該節點使用 docker 的話，下面直接示範指令

```
○ → kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
debug-pod-7bb76865bd-bwnwx   1/1     Running   0          27m

○ → pause_id=$(docker ps | grep debug-pod-7bb76865bd-bwnwx | grep pause | awk '{print $1}')

○ → docker run -it --rm --net=container:$pause_id --entrypoint /usr/sbin/tcpdump hwchiu/netutils -vvvn -i any icmp
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
04:51:08.659648 IP (tos 0x0, ttl 64, id 10959, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.13 > 8.8.8.8: ICMP echo request, id 16, seq 1656, length 64
04:51:08.682612 IP (tos 0x0, ttl 61, id 332, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.244.0.13: ICMP echo reply, id 16, seq 1656, length 64
04:51:09.661124 IP (tos 0x0, ttl 64, id 11105, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.13 > 8.8.8.8: ICMP echo request, id 16, seq 1657, length 64
04:51:09.747545 IP (tos 0x0, ttl 61, id 336, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.244.0.13: ICMP echo reply, id 16, seq 1657, length 64
c04:51:10.666374 IP (tos 0x0, ttl 64, id 11336, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.13 > 8.8.8.8: ICMP echo request, id 16, seq 1658, length 64
04:51:10.679855 IP (tos 0x0, ttl 61, id 344, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 10.244.0.13: ICMP echo reply, id 16, seq 1658, length 64
```

1. 先知道`運行於當前節點上`的 pod 名稱
2. 透過 docker ps 找到該 pod 節點的對應的 `pause` 容器 ID
3. docker run 的時候使用 --net=contianer:xxxx 來掛載相同的 network namespace
4. 運行起來的 docker 跑個 tcpdump 即可


# 節點抓取封包

上述探討了如何於容器內去抓取應用程式的封包，如果覺得上述指令實在太麻煩，要使用 knisff 或是還要再額外起一個 docker container。那本段落就來想想，如果想要於節點上直接抓取封包的話，可以怎麼做。

於節點上去透過 tcpdump 來抓取封包，我們要思考的點其實就是
1. 我要抓哪個網卡
2. 如果封包會經過很多網卡，那每個階段的封包分別代表什麼意思


這部分必須說，我想不到一個完美的方式來解決上述問題。
不同的 CNI 設定的方式與架構完全不同，因為架構不同，要透過 tcpdump 找到精準的網卡方式完全不同。
以下就針對 Flannel 以及 Calico 兩種方式來示範

## Flannel
Flannel 預設使用 VXLAN 這種透過 UDP 封裝的方式來處理節點間的封包傳輸，整個架構大概如下
![](https://i.imgur.com/dqfVlGK.png)

如果對於 Flannel 的封包傳輸有興趣的，可以參考我之前撰寫的分析文[CNI - Flannel - VXLAN 封包運作篇
](https://www.hwchiu.com/cni-flannel-iii.html)，該文章中很仔細的介紹整個封包傳輸的過程。

今天這篇文章我們不講太細，改用下列這種簡單的圖片來看架構
![](https://i.imgur.com/SboSvWB.png)

1. Flannel 的世界中，每個節點上面都會產生一個 `cni0` 的虛擬網卡，其本質是一個 Linux Bridge
2. Pod 與 Linux Bridge 會透過 veth 的方式串接彼此，一端於 Pod 裡面，通常命名為 eth0，另一端接上 cni0 上，通常命名是 veth 開頭
3. 為了滿足 VXLAN 的作法，系統上還會有 flannel.1 的網卡

簡單來說，使用 Flannel 的情況下，創建第一個 Pod 就會讓系統上產生三個網卡(vethxxx,cni0, flannel.1)，之後每創建一個新的Pod，就會產生一個全新的 vethxxxx。

```bash=
→ brctl show
bridge name     bridge id               STP enabled     interfaces
cni0            8000.f60b1ed977a3       no              vetha3a29b21
                                                        vethb33e2721
                                                        vethedce4e05
```

所以想要於節點上抓取封包時，其實可以針對 `vethxxx` 這個虛擬網卡去抓取封包，就可以抓到從容器內送出或是即將送到容器內的封包。

比較難的問題反而是，要如何知道 veth 對應到哪一個 Pod，這邊我分享一下我的作法，直接列出所有步驟
```
→ kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE      NOMINATED NODE   READINESS GATES
debug-pod-7bb76865bd-96kt4   1/1     Running   0          5m14s   10.244.0.20   k8s-dev   <none>           <none>
debug-pod-7bb76865bd-bwnwx   1/1     Running   0          62m     10.244.0.13   k8s-dev   <none>           <none>
debug-pod-7bb76865bd-dctbh   1/1     Running   0          5m14s   10.244.0.21   k8s-dev   <none>           <none>
debug-pod-7bb76865bd-kdqsl   1/1     Running   0          5m14s   10.244.0.18   k8s-dev   <none>           <none>
debug-pod-7bb76865bd-n8xgr   1/1     Running   0          5m14s   10.244.0.19   k8s-dev   <none>           <none>

→ ping 10.244.0.21 -c1
PING 10.244.0.21 (10.244.0.21) 56(84) bytes of data.
64 bytes from 10.244.0.21: icmp_seq=1 ttl=64 time=0.132 ms

--- 10.244.0.21 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.132/0.132/0.132/0.000 ms

→ arp -na | grep 10.244.0.21
? (10.244.0.21) at e2:d0:10:82:13:e1 [ether] on cni0

→ brctl showmacs cni0 | grep e2:d0:10:82:13:e1
  7     e2:d0:10:82:13:e1       no                25.35

→ brctl showstp cni0 | grep "(7)"
vethe281cd54 (7)

→ sudo tcpdump -vvv -i vethe281cd54 icmp
tcpdump: listening on vethe281cd54, link-type EN10MB (Ethernet), capture size 262144 bytes
05:29:19.930095 IP (tos 0x0, ttl 64, id 63397, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.21 > dns.google: ICMP echo request, id 490, seq 16, length 64
05:29:19.950787 IP (tos 0x0, ttl 61, id 8561, offset 0, flags [DF], proto ICMP (1), length 84)
    dns.google > 10.244.0.21: ICMP echo reply, id 490, seq 16, length 64
05:29:20.931567 IP (tos 0x0, ttl 64, id 63476, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.21 > dns.google: ICMP echo request, id 490, seq 17, length 64

```

1. 先找出你目標的 PodIP
2. 該節點上先 ping 一次該 PodIP
3. 透過 arp 看一下目標 Pod 裡面網卡的 MAC Address 是多少 (e2:d0:10:82:13:e1)
4. 透過 brctl showmacs cni0 來觀察，到底該 MAC Address 實際上在 cni0 這個 Linux Bridge 上的 port number 是多少(範例是7)
5. 透過 brctl showstep cni0 來觀察到底 `7` port 對應到的網卡是誰，範例中可以查到 `vethe281cd54`
6. 最後直接針對該網卡去 tcpdump 即可

註: 這邊不會解釋為什麼我這麼做，畢竟牽扯到一些網路運作原理

這邊抓到的是容器進出後的封包，如果想要抓到 VXLAN 包裝的封包，那步驟就會更複雜，這邊就不探討。


## Calico

Calico 並不使用 VXLAN 這種封包封裝的協定來轉發封包，取得代之的則是透過各種 routing 的方式來轉發，也因為這個特性使得要找到對應的網卡會簡單非常多。

這邊就不介紹 Calico 的架構，直接用一個簡單範例看看如何針對 Calico 的方式找到對應的虛擬網卡

```bash=
→ kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP                NODE      NOMINATED NODE   READINESS GATES
debug-pod-554f8fb4b4-4wbz5   2/2     Running   0          6m16s   192.168.252.196   k8s-dev   <none>           <none>
debug-pod-554f8fb4b4-dg2wd   2/2     Running   0          6m16s   192.168.252.194   k8s-dev   <none>           <none>
debug-pod-554f8fb4b4-hfj46   2/2     Running   0          6m16s   192.168.252.193   k8s-dev   <none>           <none>
debug-pod-554f8fb4b4-qw9g5   2/2     Running   0          6m16s   192.168.252.195   k8s-dev   <none>           <none>
debug-pod-554f8fb4b4-v5v9t   2/2     Running   0          6m16s   192.168.252.197   k8s-dev   <none>           <none>

→ ip route | grep 192.168.252.193
192.168.252.193 dev cali60337b1a8c1 scope link

→ sudo tcpdump -vvv -i cali60337b1a8c1 icmp
tcpdump: listening on cali60337b1a8c1, link-type EN10MB (Ethernet), capture size 262144 bytes
05:52:52.146265 IP (tos 0x0, ttl 64, id 63515, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.252.193 > dns.google: ICMP echo request, id 145, seq 138, length 64
05:52:52.171319 IP (tos 0x0, ttl 61, id 64009, offset 0, flags [DF], proto ICMP (1), length 84)
    dns.google > 192.168.252.193: ICMP echo reply, id 145, seq 138, length 64
05:52:53.147177 IP (tos 0x0, ttl 64, id 63672, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.252.193 > dns.google: ICMP echo request, id 145, seq 139, length 64
05:52:53.161166 IP (tos 0x0, ttl 61, id 64014, offset 0, flags [DF], proto ICMP (1), length 84)
    dns.google > 192.168.252.193: ICMP echo reply, id 145, seq 139, length 64
```


1. 先找到目標的 PodIP
2. 透過 ip route 來看看系統上會怎麼轉發這個封包，理論上可以直接找到該 IP 應該要送到哪個網卡
3. 直接對該網卡 `cali60337b1a8c1` 聽封包即可


# 結論

網路好難


# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

詳細可以參閱
矽谷年線上學院
https://course.hwchiu.com

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
