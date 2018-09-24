---
title: '[Kubernetes] DNS Setting When DnsPolicy Is Default'
keywords: 'kubernetes,DNS,Linux,k8s'
abbrlink: 48875
date: 2018-08-12 04:31:52
tags:
  - Kubernetes
  - Linux
  - DNS
  - Docker
description: 本文跟大家分享一個在實際部屬上遇到的問題，在不同的環境下，採用 dnsPolicy:Default 設定的 Kubernetes Pod 裡面所設定的 DNS Server 卻是完全不同的. 根據實際研究與觀察後，發現這個數字並不是單單的依靠 kubernetes 去處理，實際上也跟 Docker 本身如何去設定容器的 dns 也有關係，這部分就包含了宿主機的 /etc/resolv.conf 以及宿主機上 Docker 運行時的參數. 本文會先介紹這個問題，並且分享解決問題的思路以致於最後可以得到這個結論。

---

# Preface
此篇文章是 Kubernetes Pod-DNS 系列文章第二篇
此系列文會從使用者的用法到一些問題的發掘，最後透過閱讀程式碼的方式去分析這些問題

相關的文章連結如下
- [[Kubernetes] DNS setting in your Pod](https://www.hwchiu.com/kubernetes-dns.html)
- [[Kubernetes] DNS Setting with Dockerd(原始碼分析上)](https://www.hwchiu.com/kubernetes-dns-iii.html)
- [[Kubernetes] DNS Setting with Dockerd(原始碼分析下)](https://www.hwchiu.com/kubernetes-dns-iiii.html)

# 正文

在前篇文章
[[Kubernetes] DNS setting in your Pod](https://www.hwchiu.com/kubernetes-dns.html) 中已經詳細介紹了如何針對 `Pod` 去設定自己想要的 `DNS` 規則。

但是最近遇到一個有趣的狀況，當 `Pod` 內設定其 `dnsPolicy` 為 `Default` 時，則 `Pod` 內 `/etc/resolv.conf` 的數值卻會因為系統上面不同的設定而有不同的結果。

根據研究與實驗觀察後，這些東西最後會跟兩個東西有很密切的關係，分別是運行節點上 `/etc/resolv.conf` 內的資料以及該機器上 `dockerd` 運行的參數有關

直接先講結論
`kubernetes` 會先嘗試使用節點上 `/etc/resolv.conf` 的資料，但是若發現 `/etc/resolv.conf` 是空的，這時候就會去依賴 `dockerd` 幫忙產生的 `/etc/resolv.conf`

![Imgur](https://i.imgur.com/zzTxjSY.png)

針對這個問題接下來會分成兩篇文章來解釋
其中本篇是上篇，主軸在於介紹問題，並且透過實驗觀察結果進行歸納。
之後會有下篇，比較硬派一點，直接透過觀察原始碼的方式來驗證本篇的觀察結果


# 環境版本
- docker:
    - 17.06.2-ce, build cec0b72
- kubernetes: 
    - v1.10.0
- os: 
    - Ubuntu 16.04, Linux 4.4.0-128-generic

# 問題描述

首先，我觀察到這個問題主要是在不同的`Kubernetes` 集群中，我發現我自己部署的 `Deployment/Pod` 某些情況下卻沒有辦法解析外部的 DNS 名稱，譬如 `google.com`.
> 這些 `kubernetes` 集群可能是採用不同方式安裝的，如 kubespray, kubeadm
 

這些 `Pod` 都採用預設的 `DNS` 設定，所以都會採用 `ClusterFirst` 的機制讓 `kube-dns` 來處理這些 `DNS` 請求。

為了釐清這個問題，我就開始針對 `kube-dns` 裡面的 `dnsmasq` 這個容器進行研究。

我研究的方向是 `dnsmasq` 容器內 `/etc/resolv.conf` 的資料，我觀察到在不同的安裝環境下，`dnsmasq` 內 `/etc/resolv.conf` 內的資料有很多種可能性

以我自己的環境下，我看到不同 `kubernetes` 集群內 `kube-dns` 裡面 `dnsmasq`  此容器內的 `/etc/resolv.conf` 有下列變化

```shell=
root@node-1:~$ kubectl -n kube-system exec  kube-dns-5466774c4f-r9k4w cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local
nameserver 10.233.0.3
options ndots:2 timeout:2 attempts:2
```

```shell=
root@node-1:~$ kubectl -n kube-system exec  kube-dns-5466774c4f-r9k4w cat /etc/resolv.conf
8.8.8.8
```

```shell=
root@node-1:~$ kubectl -n kube-system exec  kube-dns-5466774c4f-r9k4w cat /etc/resolv.conf
10.5.23.1
```

# 思路
首先 `dnsmasq` 這個容器本身採用的是 `dnsPolicy:default` 這個選項來操作, 所以我認為這個部分應該會跟該節點本身的 `/etc/resolv.conf` 有關

所以進行了下列兩個實驗
1. /etc/resolv.conf 裡面有資料
2. /etc/resolv.conf 裡面沒有資料

# 觀察結果
## 維持 /etc/resolv.conf

首先觀察到，在所有的集群內，只要 `/etc/resolv.conf`  有資料的話，則 `dnsmasq` 的 `/etc/resolv.conf` 都會是一致的

![Imgur](https://i.imgur.com/h8cHkH2.png)

但是接下來若將 `/etc/resolv.conf` 給清空，這時候不同集群表現出來的結果卻完全不同了。


## 清空 /etc/resolv.conf
第一種案例如下，`dnsmasq` 內的則是自動的被捕上了 `8.8.8.8` 以及  `8.8.4.4`

![Imgur](https://i.imgur.com/Ms0wDUs.png)

第二種案例如下，`dnsmasq` 內的則是自動的被捕上了跟 `kubernetes kube-dns` 有關的資訊，這些資料看起來就跟 `dnsPolicy:clusterFirst` 完全一樣

![Imgur](https://i.imgur.com/owfBHyw.png)

# 歸納結果

總和以上原因，目前可以至少知道，只要 `/etc/resolv.conf` 有資料的話， `dnsmasq` 內的 `/etc/resolv.conf` 就會與該資料一致。

但是若 `/etc/resolv.conf` 沒有資料的話， 則 `dnsmasq` 內的 `/etc/resolv.conf` 目前卻出現兩種可能性。


# 思路(二)

接下來為了釐清這個問題，就開始認真的翻文件以及相關的程式碼，最後終於找到了影響的原因
> 相關的程式碼會在下篇文章解釋一切的來龍去脈

另外一個會影響數值的就是 `dockerd` 本身啟動的參數

根據上述兩個有差異的 `kubernetes` 集群去分析，發現這兩個 `kubernetes` 集群內本身運行的 `dockerd` 有參數上的差異


其中一個是非常乾淨，單純的設定連線資訊，另外一個則是設定了很多`DNS` 的數值
如下圖
![Imgur](https://i.imgur.com/dAPbB65.png)

而且可以發現這些參數的數值都跟上述 `dnsmaq` 裡面的數值完全一致

![Imgur](https://i.imgur.com/owfBHyw.png)


# 歸納結果

目前觀察的結果，當 `kubernetes` 創見 `Pod` 且 `dnsPolicy=default` 時，其內部容器 `/etc/resolv.conf` 的數值會受到兩個參數影響

1. 該節點上本身的 /etc/resolv.conf
2. 該節點上 dockerd 運行的 dns 參數


| node\dockerd | 有設定 DNS | 沒設定 DNS| 
| -------- | -------- | -------- |
| 有數值     | node     | node     |
| 沒有數值  | dockerd     | 8.8.8.8     |

基本上只要節點上的 `/etc/resolv.conf` 有資料，就直接採用
若節點上的 `/etc/resolv.conf` 沒有資料，則會根據 `dockerd` 本身有沒有額外的 `DNS 參數` 來決定，若有則使用，沒有則採用 `8.8.8.8` 以及 `8.8.4.4`


將上述的結果用更完整的說法就是
`kubernetes` 會先嘗試使用 `/etc/resolv.conf` 的資料，但是若發現 `/etc/resolv.conf` 是空的，這時候就會去依賴 `dockerd` 幫忙產生的 `/etc/resolv.conf`

