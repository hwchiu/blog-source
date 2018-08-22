---
title: '[Kubernetes] What Is Service?'
keywords: 'Kubernetes,Service,Linux,k8s'
abbrlink: 34168
date: 2018-08-18 17:08:45
tags:
  - Kubernetes
  - Linux
description:
---

本文章是屬於 `kubernetes` service 系列文之一，該系列文希望能夠與大家討論下量兩個觀念
1. 什麼是 `Kubernetes Service`, 為什麼我們需要它？ 它能夠幫忙解決什麼問題
2. `Kubernetes Service` 是怎麼實現的?， 讓我們用 Iptables 來徹徹底底的理解他

相關文章:
[[Kubernetes] How To Implement Kubernetes Service(i)](https://www.hwchiu.com/kubernetes-service-ii.html)


本篇文章偏向介紹，要跟大家討論為什麼在 `kubernetes` 集群內需要有 `service` 的服務，這個服務能夠解決什麼問題
以及最後透過實際範例跟大家介紹如何使用

<!--more-->

## What Problems

我們都知道 `kubernets` 集群擁有非常強大的功能，可以讓管理者透過 `Deployment` 很輕鬆的去部署各式各樣的服務，譬如 `Web Server`, `database`,`monitor` 等各式各樣的服務。

舉一個使用場景來説，假設我今天在集群內部署了`MongoDB` 作為我的資料庫服務，同時我其他的應用程式需要透過網路跟該 `MongoDB` 進行連線存取。
此外，此情境中使用了 `MongoDB` 集群的概念，讓`MongoDB`本身同時會有多個`Pod`運行在集群之中

上述的情境如下圖所示，圖中紅色顯設的則是 `MongoDB` 所對應的 `Pod`, 而每個 `Pod` 底下都有一個屬於自己的 `IP` 地址
![Imgur](https://i.imgur.com/PBRRRA9.png)

在這種情況下，我們自己邏輯業務的應用程式則是用綠色區塊顯示，在該應用程式內，為了要跟 `mongo` 進行連線存取，則必須要知道這些 `mongo` 應用程式的 `IP` 地址

但是在 `kubernetes` 集群中，要是這些 `Mongo` 對應的容器發生錯誤或是因為其他問題而發生的容器停止然後重啟的事件
會導致這些容器之後都會擁有一個完全不同的 `IP` 地址
{% note danger %}
事實上，如果夠熟悉 CNI 與 IPAM 的開發者，其實是有辦法讓Pod擁有固定 `IP` 地址的
{% endnote %}

如下圖所示。
在這種情況下，我們的應用程式要怎麼知道這些 `IP` 已經改變? 如何應因這些改變而修正我們應用程式連線的對象?
![Imgur](https://i.imgur.com/lhICs7Q.png)

## How To Solve
以前撰寫應用程式的時候，針對目標的`IP`地址根據情境會有不同的處理方式，可能會寫死在應用程式裡面，也有可能會透過設定檔案來讀取，也有可能會透過 `DNS` 解析的方式來處理這個問題。
然後不是每個人都會架設 `DNS` 伺服器來處理。

為了解決這個問題， `Kuberntes Service` 就是這個問題的救命仙丹。
`service` 的概念非常簡單，將整個`IP`地址與連線的方法給分層次處理。

我們使用下圖來展示這個概念
![Imgur](https://i.imgur.com/tmLI7a4.png)
首先圖示中橘色的部份就是 `Kubernetes Service` 的邏輯概念
1. 該 `Service` 本身會先綁定特定的應用程式，在此範例內就是這些 `Mongo` 的容器們，`kubernetes service` 本身會自己去追蹤並且更新對應`Mongo`容器的 `IP` 地址
2. `Service` 此外還會提供一組 `FQDN` 的名稱去供其他的應用程式使用。
舉例來說，我們的`App`可以透過這組 `FQDN` 去存取這些 `Mongo` 容器，對於 `App` 來說，只要相信這組 `FQDN` 即可，至於背後到底會對應到哪些 `Mongo` 容器，則是交由 `Service` 幫忙處理。

引入了 `Kubernetes Service`  這種架構後，我們需要部屬到 `kubernetes` 集群的服務流程如下 (有 `re-try` 機制的話順序其實不重要)
1. 部屬 `Mongo` 服務到集群(有多少個Pod都沒關係)
2. 部屬 `Kubernetes Service` 到集群，並且設定該 `Service` 連接到上述部屬的 `Mongo` 服務
3. 部屬自行的應用程式，該應用程式則用 (2) `kubernetes service` 所產生的 `FQDN` 來連線。


`kubernetes service` 本身也有不少種類可以選擇，目前總共有四種可以使用，分別是
1. ClusterIP
2. NodePort
3. LoadBalancer
4. External

其中目前大家最常用的就是 `ClusterIP` 以及 `NodePort`，所以下面介紹一下這兩者的差異。
{% note danger %}
本篇文章著重在特性與概念的介紹，背後的實作原理會等到下篇文章在來介紹與分析。
{% endnote %}

### ClusterIP
`ClusterIP` 的意思就是只有集群內的應用程式/節點可以透過該組 `FQDN` 去存取背後的服務。
在此情況下，除了透過`kubernetes`去部屬的應用程式外，預設情況下都沒有辦法透過該`FQDN`去存取，即使你直接使用了`kubernetes dns`來問到對應的`IP`地址也沒有辦法。
{% note danger %}
這邊指的是預設情況下，如果夠懂網路以及背後原理，當然還是有辦法可以從外面存取到這些服務的
{% endnote %}

### NodePort
`NodePort` 本身包含了 `ClusterIP` 的能力，此外多提供了一種能力讓`非集群`的應用程式/節點也有辦法存取集群內的應用程式。
舉例來說，我們可以部屬多個網頁伺服器，然後透過 `NodePort` 的方式讓外部的電腦(瀏覽器）來存取這些在 `kubernetes` 集群內的網頁伺服器。

由前面我們知道，`kubernetes service` 務提供的 `FQDN` 只能供`集群`內的應用程式去存取。
那要如何達到`非集群`的應用程式也能夠存取集群內的應用程式?
這邊就如同其字面`NodePort`一樣，任何`非集群`內的應用程式都可以透過存取`集群`節點上的特定`Port`轉而存取到集群內的應用服務。

{% note danger %}
詳細的運作原理留到下篇文章在好好的跟大家探討與分享
{% endnote %}

最後用一張圖片來說明 `ClusterIP` 以及 `NodePort` 兩者的關係
![Imgur](https://i.imgur.com/q0j2z4J.png)

在圖示中，紫色的`K8S App`就是所謂的集群內應用程式，而紅色的`HostApp`就是所謂`非集群`的應用程式。

- ClusterIP: 只有紫色的應用程式以及集群內的節點可以存取
- NodePort: 紫色跟紅色的應用程式都可以存取，只是存取的方式些許不同。注意的是該非集群內的應用程式可以運行在任何節點上，只要有辦法透過網路與`kubernetes`集群內集點相連即可。


## How To Use It

接下來使用[kubeDemo](https://github.com/hwchiu/kubeDemov)專案內的內容來展示一下如何使用 `ClusterIP` 以及 對應的 `NodePort` 服務。

在此範例中，我採用 `Nginx` 作為一個後端的服務，然後用 `ubuntu` 當做一個集群內的應用程式，想要透過 `Service` 的方式存取到 `Nginx`

![Imgur](https://i.imgur.com/osNqxlw.png)


首先，我們先部屬相關的應用程式`Ngnix` 以及用來測試用的 `ubuntu`
```shell=
vortex-dev:04:10:04 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl delete pod test-pod-a581017d9b1dff9dec4
pod "test-pod-a581017d9b1dff9dec4" deleted
vortex-dev:04:10:45 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl apply -f services/deployment/nginx.yml
deployment.apps/k8s-nginx created
vortex-dev:04:16:17 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl apply -f services/application/ubuntu.yml
pod/ubuntu created
```

部屬完畢後，接下來我們要來部屬相關的 `Cluster-IP` 以及 `NodePort` 兩個服務
### Deploy ClusterIP
```bash=
vortex-dev:04:29:45 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl apply -f services/service/nginx-cluster.yml
service/k8s-nginx-cluster created
```

仔細研究一下 `services/service/nginx-cluster.yml` 檔案的內容
```yaml=
apiVersion: v1
kind: Service
metadata:
  name: k8s-nginx-cluster
  labels:
    run: k8s-nginx-cluster
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: k8s-nginx
```
這邊用的是非常簡單範例
1. 這邊沒寫 Type, 預設就會是 `ClusterIP`.
1. 該 `service` 會透過 `selector` 去找名稱是 `k8s-nginx` 的`nginx`的應用服務，並且告知該應用服務是使用 `TCP:80` 去連線
2. 該 `service` 本身是名稱是 `k8s-nginx-cluster`

接下來透過 `kubectl describe` 來觀察一下該 `service`.

```bash=
vortex-dev:04:32:55 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl  describe svc k8s-nginx-cluster
Name:              k8s-nginx-cluster
Namespace:         default
Labels:            run=k8s-nginx-cluster
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"run":"k8s-nginx-cluster"},"name":"k8s-nginx-cluster","namespace":"default"}...
Selector:          run=k8s-nginx
Type:              ClusterIP
IP:                10.98.51.150
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.0.88:80,10.244.0.89:80,10.244.0.90:80
Session Affinity:  None
Events:            <none>
```

這邊先注意的就是 `Name` 以及 `Namespace`  這兩個欄位，因為該 `service` 會用 **\$Name.$Namespace** 的方式吐出一個可以使用的 `FQDN` 供其他應用程式使用
此範例中就是 `k8s-nginx-cluster.default`。

為了驗證這個情境，我們嘗試透過剛剛部屬的 `Ubuntu` 去嘗試對 `NGINX` 存取網頁看看
```bash=
vortex-dev:04:48:50 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl exec ubuntu curl -- -s k8s-nginx-cluster.default
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

非常順利的存取到網頁了，這時候如果想要從節點本身(`非集群應用程式`)去存取看看呢?
```bash=
vortex-dev:04:54:37 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$curl k8s-nginx-cluster.default
curl: (6) Could not resolve host: k8s-nginx-cluster.default
```
會發現根本連該 `FQDN` 的 `DNS` 解析都沒有辦法。

{% note  danger %}
實際上 ClusterIP 是因為 kube-dns 的關係沒有辦法解析該位置
但是若嘗試直接使用解析過後的IP位置去存取
集群內的節點透過解析後的地址是可以存取到目標的。
只是一般人不會想要直接使用該 `IP`，而是更依賴使用 `FQDN` 的方式。
{% endnote %}

### Deploy NodePort
接下來我們嘗試部屬看看 `NodePort` 的 `service`

```bash=
vortex-dev:04:29:45 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl apply -f services/service/nginx-node.yml
service/k8s-nginx-node created
```

仔細研究一下 `services/service/nginx-cluster.yml` 檔案的內容
```yaml=
apiVersion: v1
kind: Service
metadata:
  name: k8s-nginx-node
  labels:
    run: k8s-nginx-node
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: k8s-nginx
  type: NodePort
```
這邊可以觀察到
1. 特別標示該 `Type` 是 `NodePort`
2. 該 `service` 本身是名稱是 `k8s-nginx-node`

接下來透過 `kubectl describe` 來觀察一下該 `service`.

```bash=
vortex-dev:04:32:55 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$kubectl  describe svc k8s-nginx-node
Name:                     k8s-nginx-node
Namespace:                default
Labels:                   run=k8s-nginx-node
Annotations:              kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"run":"k8s-nginx-node"},"name":"k8s-nginx-node","namespace":"default"},"spec...
Selector:                 run=k8s-nginx
Type:                     NodePort
IP:                       10.99.157.45
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32293/TCP
Endpoints:                10.244.0.88:80,10.244.0.89:80,10.244.0.90:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

這邊要注意的是除了之前的 `Name/Namespace` 之外，多了 `NodePort` 的欄位出現了，這邊後面的 `32293` 就是代表可以透過任意集群節點上面的 `TCP:32293` 去存取到內部的 `Nginx` 服務器

我們直接使用 `172.17.8.100:32293` 嘗試看看
```bash=
vortex-dev:05:03:44 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
$curl 172.17.8.100:32293
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


## Summary
本章節中，我們介紹了 `Kubernetes Serive`, 為什麼需要 `Service` 以及 `Service` 如何解決我們的問題
同時介紹了常用的 `ClusterIP` 以及 `NodePort` 這兩種類型的差異以及概念
最後透過幾個簡單的範例展示下如何使用 `ClusterIP`/`NodePort` 讓我們能夠更方便的透過 `service` 去存取我們的後端服務
