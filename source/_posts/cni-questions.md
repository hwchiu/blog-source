---
title: CNI 常見問題整理
keywords: 'CNI, kubernetes, ip, pod, container network interface'
abbrlink: 31436
date: 2018-10-20 06:49:39
tags:
  - CNI
  - Docker
  - Network
  - Kubernetes
description: 本篇文章紀錄了作者這陣子以來與大家討論 CNI 時常常被問到的問題，透過對這些問題的理解可以更加深入的去學習什麼是 CNI, 以及 CNI 本身能夠能夠觸擊的功能與範圍。同時也附上一些相關的資源讓大家可以從不同角度更深入的去研究 CNI 的領域。

---


# Preface
這篇文章用來記錄這陣子以來遇到跟 CNI 有關的任何問題，大部分都是偏向使用上的用法，或是對 CNI 功能上的疑慮

# Questions
## 什麼是 CNI
1. 簡單的說，就是可以讓你的 `Container` 上網的一種方法，其中包括了讓你的 `Container` 能夠有對外存取的方法，IP 的設定甚至到 `Container` 內部關於 `DNS` 伺服器的設定都可以
2. 複雜的答案參考 [[Container Network Interface] CNI Introduction](https://www.hwchiu.com/introduce-cni-ii.html) 這篇文章，有比較詳細的介紹 `Container Network Interface(CNI)` 的概念

## 我現在使用 kubernetes, 到底要選擇哪個 CNI
1. 這個問題非常容易聽到，但是基本上沒有辦法回答，因為網路的架構太過於複雜且龐大，沒有明確的需求之前沒有辦法得到一個較好的答案。
2. 如果你只是想要 `Pod` 之間可以正常連線，沒有其他的考量的話，就選擇 `Flannel` 吧。
3. 有其他網路使用相關的需求，甚至是 `Network Policy` 的話，就要在尋求其他的 CNI 解決方案
4. 之後會再寫一篇文章來簡單介紹一下目前有哪些 `CNI` 解決方案，分別有什麼樣的特色


## 聽說 CNI 可以設定 DNS, 那為什麼我在 kubernetes 環境中這些設定都沒有被採用
1. CNI 的確有提供 DNS 的選項，但是會不會採用要依賴使用該 `CNI` 的管理系統的決策，對於 `Kubernetes` 來說, 因為已經可以透過 `Pod` 的設定檔來處理相關的 `DNS` 設定了，所以 `CNI` 本身回傳的 `DNS` 設定就忽略掉，並不會採用。
2. 可以參考這篇文章學習如何透過 DNSPolicy 來個別設定 Pod 裡面的 DNS欄位, [DNS setting in your Pod](https://www.hwchiu.com/kubernetes-dns.html)
3. 相關的 `Gihub` 討論可以參考這篇 [Unable to add custom DNS to container using flannel plugin](https://github.com/containernetworking/plugins/issues/128)

## 我有沒有辦法透過 CNI 去更彈性的設定 `Pod` 的 IP 地址
1. 在 `CNI` 裡面有 `IPAM` 相關的模組在進行 IP 地址的計算與分配，在預設的情況下，大部分的 `CNI` 都會使用 `IPAM(host-local)` 搭配使用者預先輸入的一組網段來分配 `IP` 地址給每個運行的 `Pod`.
2. 由於這邊沒有狀態紀錄的概念，所以相同 `Deployment` 產生的`Pod`每次都會拿到不一樣的 `IP` 是很正常的。
3. 如果想要做到一些特別的需求，譬如 Static IP，或是 `Container` 容器重啟可以拿到相同的 `IP` 地址，這部分目前都要依賴重新撰寫 `IPAM` 的模組來達成這個功能。

## CNI 有沒有支援 HA
1. 其實這個問題我聽到的時候也愣住了，我其實聽不太懂這個問題，所以忽略吧
2. 認真討論的話這個東西跟 CNI 架構無關，是你採用的那套 CNI 本身有沒有幫你實現網路相關的 HA 功能

## 我想要在 Pod 有多張網卡甚至是同時運行許多現有的 CNI，可以怎麼做?
1. 對於 Kubernetes 來說對於每個 `Pod` 都只會呼叫一次 `CNI`. 所以如果想要在一個 `Pod` 裡面有多張網卡或是執行多次 CNI 的話，有兩種方式
  - 自行重新撰寫一個 CNI, 當該 CNI 被呼叫將相關的參數傳遞到想要呼叫的所有 CNI 來達成目的
  - 使用現有的解決方案來達成上述目標
2. 目前我自己知道能夠處理這個問題的 CNI 專案有三個，如果有人看到更多的歡迎告訴我
  - [Intel Multus](https://github.com/intel/multus-cni)
  - [Huawei Genie](https://github.com/Huawei-PaaS/CNI-Genie)
  - [Nokia DANM](https://github.com/nokia/danm/blob/master/README.md#introduction
)
3. 這三個解決方案裡面我個人比較熟悉的是偏向 Intel Multus, 其功能跟 Huawei Genie 比較類似，都是專注於提供 `CNI Chain` 的概念，可以透過設定檔去描述多個 `CNI` 並且依序執行這些 `CNI` 來幫 `Pod` 提供網路功能
4. 相對於上述兩者而言， `Nokia DANM` 提供的功能更廣大，`CNI Chain` 只是其中一項功能而已，有興趣的朋友可以在閱讀其官方說明文件

## 我要如何知道我的 kubernetes 到底現在是用哪個 CNI
1. 先確認你的 `kubelet` 是採用 `CNI`，這部分可以透過觀察 `kubelet` 的參數來得知
```bash=
root      1173  3.3  2.6 634128 105920 ?       Ssl  Oct16 164:30 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf 
--kubeconfig=/etc/kubernetes/kubelet.conf 
--config=/var/lib/kubelet/config.yaml 
--cgroup-driver=cgroupfs 
--cni-bin-dir=/opt/cni/bin 
--cni-conf-dir=/etc/cni/net.d 
--network-plugin=cni
```
有三個跟 `CNI` 有關的參數，其中兩個分別描述 `執行檔` 以及 `設定檔` 的位置
2. 接下來到 ${cni-conf-dir} 的位置去看， `kubernetes` 會從裡面根據`字母排序`的方式找到第一個`設定檔案`來使用。
```bash=
hwchiu@k8s-dev:~$ ll /etc/cni/net.d/total 12
drwxr-xr-x 2 root root 4096 Oct  8 15:10 ./
drwxr-xr-x 3 root root 4096 Oct  6 13:52 ../
-rw-r--r-- 1 root root   92 Oct  8 15:10 10-flannel.conf
```

## 要如何簡單偵錯 CNI 是否有運作正常
1. 這部分其實滿難的，以 `Kubernetes` 來說，除了每個 `CNI` 解決方案本身是否有相關的 `Daemon` 或是其他方式可以觀察 Log 之外，剩下的就是依賴 `kubelet` 本身呼叫 `CNI` 的結果。這部分可以透過 `journalctl` 來觀察 `kubelet` 的輸出訊息(前提是你有用 systemd) 控管
2. 使用 `sudo journalctl -xe` 觀察的話，可以專心觀察 `cni.go` 的輸出訊息

## 我的 `Pod` 網路有問題，我可以怎麼辦
1. **放大招**,砍掉 Kubernetes, 重新安裝
2. 先看看自己使用的是哪套 `CNI`, 然後搜尋看看那個 `CNI` 有沒有相關的 `Issue`
3. 手動`Debug`, 通常我遇到這種情況我會採取下列步驟
  - 先透過 `nslookup` 等工具觀察 `DNS` 解析是否正常。
  - 透過 `ping/telnet` 等工具觀察連線是否正常
  - 透過 `ip route/route` 觀察路由目標是否正常
  - 透過 `tcpdump` 抓取封包，觀察封包的流向
  - 透過 `iptables` 肉眼觀察是否多餘或是缺少特定的規則導致連線起不來
4. 上述 `debug` 的步驟有些會牽扯到 `Host` 主機，有些會牽扯到 `Container`，基本上要對整個網路跟封包傳輸的流程要有些概念才會比較知道自己在做什麼，以及該怎麼解讀看到的資訊。這部分需要時間與經驗去學習

## Kubernetes 裡面我可以每個節點都用不同的 `CNI` 嗎
1. 可以，因為 `CNI` 是跟者 `kubelet` 一起跑的，所以每台節點都是獨立設定，互相不影響
2. 在 `CNI` 的解決方案中，有的方案會讓管理者用起來很方便，不需要每台設定，譬如 `Flannel` 等則是會透過 `DaemonSet` 的方式自動幫每個節點安裝對應的 `CNI Config`。
3. 然而也有些 `CNI` 本身的使用上就需要特別去注意每台機器的資訊，特別是跟硬體網卡資訊有關聯的 `CNI` 解決方案，譬如 `SR-IOV/DPDK/Host-Device`
4. 假設每個節點都用不同的 `CNI`, 那你要擔心的可能是不同節點之間的 `Pod` 會不會因為 `CNI` 分配不同的 `IP` 位址，同時有沒有對應的 `路由規則` 來幫忙處理不同 `Pod` 之間的網路連線

## 我好想寫程式呀！ 有沒有辦法自己撰寫 CNI
1. CNI 本身其實滿容易撰寫的，只要瞭解其基本概念並且知道自己想要做什麼就差不多可以完成了
2. CNI 本身沒有限定任何語言實現，只要可以提供一個 `Binary` 執行檔即可
3. 如果對於用 `Golang` 撰寫一個簡單的 `Bridge CNI` 可以參考下列文章 [[Container Network Interface] Implement Your CNI In Golang](https://www.hwchiu.com/introduce-cni-iii.html)

# Summary
`Container Network Interface(CNI)` 本身概念不難，困難的其實都是網路本身，封包怎麼傳輸，不同節點之間怎麼路由，Overlay Network怎麼做，`Network Policy` 怎麼實現等諸多的網路問題實際上才是最令人頭疼的地方。
目前沒有看到任何一套 `CNI` 可以滿足所有的需求，所以在選擇 `CNI` 的部分還是要謹記自己的需求，從自己的需求出發，看看有沒有現成的解決方案可以採用，如果都沒有則需要評估是否需要自己開發一套 `CNI` 來實現所缺的功能。
