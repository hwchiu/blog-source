---
title: Introduction to Kubernetes Ingress (Nginx)
keywords: 'Kubernetes,Ingress, Nginx, Example'
abbrlink: 12578
date: 2019-01-06 07:44:23
tags:
  - Kubernetes
  - Network
description: Kubernetes 本身提供了非常多好用且方便的資源，其中專於網路存取的部分除了 Service 是常用元件以外， Ingress 也是一個熱門且重度發展的項目。本文詳細的介紹了 Kubernetes Ingress 的架構並且實際以 Nginx Ingress Controller 作為一個範例去架設測試環境。該測試環計中會實驗基於 Host 與 Path 等不同方式的路由設定，並且解釋該測試環境的運作原理，讓讀者更能夠直接瞭解到 Ingress 的運作方式。

---


# Preface
為了能夠讓使用者能夠更直接順利的存取到 `kubernetes` 眾多運行的 `Pods`, `kubernetes` 花了很大把的精力在網路架構與使用這方面，之前已經詳細介紹過 `Kubernetes Service` 的[用法以及原理](https://www.hwchiu.com/kubernetes-service-i.html)
這篇要來跟大家介紹另外一個相輔相成且好用的概念，也就是所謂的 `Ingress`. 透過 `Ingress` 我們能夠提供一些更方便的伺服器存取，不論是基於 `URL` 的存取導向，亦或是簡化整個 `SSL` 憑證部署的方式
都能夠簡單地完成。 為了能夠順利使用 `SSL` 憑證，我們也可以搭配 [Cert-Manager](https://www.hwchiu.com/cert-manager.html) 來進行憑證的處理，並且將其與 `Ingress` 給整合。

本文主要會從 `Ingress` 的基本概念出發，介紹其基本架構並且從最常用的 `Ingress Nginx` 作為一個使用範例，來介紹實際上整個 `Kubernetes` 集群內是如何運作的

# Introduction
`Kubernetes` 有一個非常有趣且迷人的地方，就是大量的抽象化，部分的功能則是公開其介面，依賴第三方廠商自行實現該介面來提供此功能。
譬如以前有提過的 `Network Policy` 就是這樣的一種概念，不同的`網路解決方案提供者`可以針對自己的應用特性來實現對應的 `Network Policy` 進而提供多一層的網路保護。
而 `Ingress` 也是一個類似的架構，接下來會仔細介紹一下其架構以及如何使用。最後則是會使用 `Nginx Ingress Controller` 當作一個範例來解釋整個運作


## Architecture
`Kubernetes` 的`Ingress` 裡面我們可以設定一些如何轉發封包的選項，範例如下


```yaml=
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
  name: nginx-demo
  namespace: default
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: nginx
          servicePort: http
      - path: /v2
        backend:
          serviceName: nginx-v2
          servicePort: http
  - host: note.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: jupyter
          servicePort: http
```

`Ingress` 內我們可以針對 `Host` 或是 `Path` 不同的選項來決定該封包要怎麼轉發。以上述範例來說，我們希望達到的是
1. 如果看到的是 `note.example.com` 可以送給特定的 `service`
2. 如果看到的是 `nginx.example.com` ，則根據後面的 `path (v1/v2)` 來決定最後要怎麼轉發。

這邊要特別注意的是，對於 `Kubernetes` 來說， `Ingress` 物件本身只有描述的功用，實際上並不會真的把使用者所描述與敘述的功能給實現完畢，這部分需要依賴剩下的元件來補足。

接下來我們使用下列這張圖示來解釋一下一個完整 `Ingress` 的架構。

![Imgur](https://i.imgur.com/DSlMKh9.png)

圖中標示為 `Ingress Resource` 的元件就是使用者們透過 `Ingress Yaml` 去描述預期行為的設定檔，也就是上圖的部分。
綠色的 `Backend server` 則是後端不同類型的服務器，使用者會預期 `Ingress` 可以根據 `Host/Path` 等不同的規則將對應的封包轉發到後端真正服務的 `Backend Server`.

接下來真正重要的就是 `Ingress-Controller` 以及 `Ingress-Server`.
`Ingress-Server` 普遍上來說，就是一個能夠接受 `HTTP/HTTPS` 連線的網路伺服器，以本篇文章來說就是 `Nginx`.
過往的使用經驗上，我們的確可以透過 `nginx.conf` 的方式來設定 `nginx server`. 來達到根據不同的情況來決定不同的封包轉發等行為。
但是在 `kubernetes ingress` 的架構下，使用者並不一定熟悉 `nginx.conf` 的格式與撰寫，而熟悉的只有 `Kubernetes Ingress` 的格式。
在此條件下，我們需要一個轉換者，該轉換者能夠將 `Ingress Resource` 的設定轉換成 `Ingress-Server (Nginx)` 所能夠處理的格式。
這個角色也就是所謂的 `Ingress-Controller`.

將上述的設定與使用流程以順序來看
1. 使用者透過 `yaml` 部署 `ingress` 設定到 `kubernetes` 裡面
2. `Ingress-Controller` 偵測到 `Ingress Resource` 的更動，讀取該設定後產生對應的 `Nginx.conf` 供 `Ingress-Server` 使用
3. 外部使用者嘗試存取服務，該封包會先到達 `Ingress-Server(Nginx)`.
4. `Ingrss-Server(Nginx)` 根據 `nginx.conf` 的設定決定將該封包轉發到後段的服務器 `backend server`.

從上述的概念來說，我們可以簡單歸納一下 `Ingress` 的架構
1. `Kubernetes` 本身只提供一個統一的 `Ingress` 介面，本身不參與該介面的實作
2. 服務提供者本身必須要實現 `Ingress-Server` 以及 `Ingress-Controller` 這兩個元件將使用者描述的抽象概念轉換成實際上可以運作的設定

所以可以看到目前現實上有非常多的 `Ingress` 實作，不論是 `Traefik `, `Kong`, `Nginx` 甚至是各個公有雲(GKE/AKS/EKS)都有跟自行架構更加整合的實現。

關於 `Ingress` 更多的概念可以參考[官方文件](https://kubernetes.io/docs/concepts/services-networking/ingress/#types-of-ingress)

### Annotation
有實際上使用過 `Ingress` 的玩家會發現在 `Yaml` 內有各式各樣的 `annotation` 要使用，譬如
```yaml=
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
```

首先，根據上述的架構概念，我們可以知道 `Ingress-Controller` 會去讀取 `Ingress` 的設定，然後來進行後續的動作。
假想一個情況，系統內同時有多個 `Ingress-Controller` 的實現，使用者要如何指派該 `Ingress` 要使用哪一個 `Ingress-Controller` 來使用?

再者個情況下，我們可以透過 `annotations` 的方式來加註一些額外的資訊，當然這些資訊不是標準的，反而是各個 `Ingress-Controller` 自行決定看到什麼樣的資訊來進行什麼處理。

此外，不同的 `Ingress-Server` 提供的功能與使用方式都不同，基本上 `Ingress Resource` 很難有一個完美的介面來滿足所有的實現，因此大部分的情況下，不同的 `Ingress-Controller/Ingress-Server` 都會要求使用者在 `Annotation` 的部分使用特定的字眼來描述額外的功能，譬如
```yaml=
    nginx.ingress.kubernetes.io/rewrite-target: /
```

所以在選擇 `Ingess` 的使用上，遇到任何問題的時候，如果是 `Ingress Resource` 的介面問題，則可以尋求 `Kubernetes` 官方文件的幫助，如果是更細緻的需求，則該查詢 `Ingress-Controller` 的說明文件，看看自己的需求與相對應的設定是否有辦法完成。

## Compare
跟 `Kubernetes Service` 比較起來，兩者都在提供便捷的`網路存取`服務
`Server` 針對的單位主要是 `Pod(Container)`, 提供一個更方便的方式讓用戶端可以不用在意後端 `Pod(Container)` 的真實 `IP` 地址。
而 `Ingress` 目前的使用上更偏向是 `HTTP/HTTPS` 的應用，在上述的 `Service` 上搭建一層更方便的服務，可以根據 `Host(NameBasd Virtaul Hosting)` 或是 `Path(Fanout)` 來決定後續真正轉發的對象，而該對象則是不同後端服務所搭建起來的 `Service`.

因此在使用上，這兩者並沒有誰取代誰的問題，反而是根據需求來使用，大部分情況下都是互相整合來提供更方便與好用的功能。

# Example
上述已經介紹完關於 `Ingress` 的基本概念，接下來要使用 `Nginx` 作為 `Ingress-Server` 來實際搭建一個 `Ingress` 的範例。



接下來的所有範例文件都可以在 [KubeDemo](https://github.com/hwchiu/kubeDemo/tree/master/ingress) 內找到對應的檔案。


該範例的架構圖如下
![Imgur](https://i.imgur.com/8pBjUoc.png)

我們會在 `Kubernetes` 裡面進行下列部署
1. 搭建一個基於 `Nginx` 的 `Ingress-Controller/Ingress-Server`，
2. 透過 `Deployment` 部署三套不同的後端服務，分別是 `Jupyter Notebook` 以及兩個有者不同 `Index.html` 的 `Nginx Server`.
3. 透過三個不同的 `kubernetes service` 將上述的 `Deployment` 包裝起來提供更方便的存取功能
4. 部署對應的 `Ingress`, 希望可以完成
  - 存取 `note.example.com` 會存取到 `jupyter notebook`
  - 存取 `nginx.example.com/v1` 會存取到 `nginx`
  - 存取 `nginx.example.com/v2` 會存取到 `nginx-v2`


接下來我們就要依據上述的概念進行相關檔案的部署。

我的測試環境是基於 `MAC` 上透過 `Vagrant` 創建一個 `UbuntuOS` 並且使用 `Kubeadm` 實現的一個小型 `Kubernetes` 叢集。


## Nginx Controller

首先我們要先部署 `Nginx Controller` 到 `kubernetes` 叢集內，詳細的安裝方式可以參閱其[官網](https://kubernetes.github.io/ingress-nginx/deploy/)

首先我們要先安裝相關的資源，譬如 `RBAC` 以及相關的 `Deployment`.

```yaml=
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

接下來為了讓外界的服務可以存取到該 `Nginx Server (Ingress-Server)`，這邊會根據你的機器環境而有所不同。
以我 `Baremetal` 的環境，我需要部署下列的資源，透過 `Kubernetes Service NodePort` 的方式讓我的 `Nginx Server` 可以被外界存取

```yaml=
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```

上述的部屬都會安裝到 `ingress-nginx` 的 `kubernetes namespace` 上，所以透過下列的指令觀察安裝的情形
```bash=
$ kubectl -n ingress-nginx get all
NAME                                           READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-d88dbf49c-9b6td   1/1     Running   0          23h

NAME                    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx   NodePort   10.111.134.97   <none>        80:32663/TCP,443:31309/TCP   1d

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller   1         1         1            1           1d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-controller-d88dbf49c   1         1         1       1d
```

## Outside Internet Access
為了從外部`MAC`的瀏覽器進行測試，我對機器上的 `/etc/hosts` 進行了相關的修改, 加上了下列的資料，讓我可以透過相關的設定來存取該 `NodePort` 的 `Nginx(Ingress-Server)`.
```bash=
172.17.8.101 nginx.example.com note.example.com
```
`172.17.8.101` 是我`VM(Ubuntu)`的`Virtaul IP address`.

## Backend Servers
### Jupyter
基本上需要的就是 `Deployment` 配上一個對應的 `Service` 即可
詳細的請參閱 [KubeDemo](https://github.com/hwchiu/kubeDemo/blob/master/ingress/jupyter.yml)
```bash=
kubectl apply -f https://raw.githubusercontent.com/hwchiu/kubeDemo/master/ingress/jupyter.yml
```

### Nginx
類似上述 `Jupyter` 的安裝流程，但是為了客製化 `index.html` 的內容，會額外部署一個 `configMap` 來產生不同的內容
```bash=
kubectl apply -f https://raw.githubusercontent.com/hwchiu/kubeDemo/master/ingress/nginx.yaml
kubectl apply -f https://raw.githubusercontent.com/hwchiu/kubeDemo/master/ingress/nginx2.yaml
```

這些應用程式都會部署到 `default` 這個 `namespace`，所以可以用下列指令確保部署正確
```bash=
kubectl -n default get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/jupyter                     1/1     Running   0          15h
pod/nginx-7dd9f89db4-tvfkk      1/1     Running   0          15h
pod/nginx-v2-5c45597f57-746p8   1/1     Running   0          15h

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/jupyter      LoadBalancer   10.106.190.88   <pending>     80:32444/TCP   15h
service/kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        89d
service/nginx        ClusterIP      10.110.237.87   <none>        80/TCP         15h
service/nginx-v2     ClusterIP      10.103.43.44    <none>        80/TCP         15h

NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx      1         1         1            1           15h
deployment.apps/nginx-v2   1         1         1            1           15h

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-7dd9f89db4      1         1         1       15h
replicaset.apps/nginx-v2-5c45597f57   1         1         1       15h
```

上述服務完畢後，先用 `curl` 針對三個 `service` 的 `ClusterIP` 去確認服務有正常起來

```bash=
$ curl 10.110.237.87
Nginx V1
$ curl 10.103.43.44
Nginx V2
$ curl 10.106.190.88/tree
....
```

### Deploy Ingress
上述相關的服務都部署完畢後，接下來就要部署 `Ingress` 物件進去，我們期望的行為是 `Nginx Controller` 能夠讀取這個 `Ingress` 的物件並且產生對應的 `nginx.conf` 供 `Nginx Server(Ingress-Server)` 使用.
所以先來看一下對應的 `Ingress Resource`
```yaml=

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: nginx-demo
  namespace: default
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: nginx
          servicePort: http
      - path: /v2
        backend:
          serviceName: nginx-v2
          servicePort: http
  - host: note.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: jupyter
          servicePort: http
```

該物件部署完畢後，透過下列指令觀察部署結果
```bash=
$ kubectl get ingress
NAME         HOSTS                                ADDRESS   PORTS   AGE
nginx-demo   nginx.example.com,note.example.com             80      15h
```

### Check Nginx
接下來我們將直接進入到 `Nginx Controller` 去觀察一下是否有對應的 `nginx.conf` 被產生
```bash=
$ kubectl -n ingress-nginx exec -it $(kubectl -n ingress-nginx get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath="{.items[0].metadata.name}") bash
www-data@nginx-ingress-controller-d88dbf49c-9b6td:/etc/nginx$
www-data@nginx-ingress-controller-d88dbf49c-9b6td:/etc/nginx$ grep  "example.com" nginx.conf
        ## start server nginx.example.com
                server_name nginx.example.com ;
        ## end server nginx.example.com
        ## start server note.example.com
                server_name note.example.com ;
        ## end server note.example.com
www-data@nginx-ingress-controller-d88dbf49c-9b6td:/etc/nginx$ grep  "v1" nginx.conf
        ssl_protocols TLSv1.2;
                location ~* "^/v1\/?(?<baseuri>.*)" {
                        set $location_path  "/v1";
                        rewrite "(?i)/v1/(.*)" /$1 break;
                        rewrite "(?i)/v1$" / break;
www-data@nginx-ingress-controller-d88dbf49c-9b6td:/etc/nginx$ grep  "v2" nginx.conf
                location ~* "^/v2\/?(?<baseuri>.*)" {
                        set $service_name   "nginx-v2";
                        set $location_path  "/v2";
                        set $proxy_upstream_name "default-nginx-v2-http";
                        rewrite "(?i)/v2/(.*)" /$1 break;
                        rewrite "(?i)/v2$" / break;
```

到這邊為止，基本上的一切都順利設定完畢了，接下來就可以開啟瀏覽器嘗試去瀏覽看看.

### Access the Nginx Server
因為我們的 `Nginx Server` 是基於 `NorePort` 的方式來供對外存取，所以我們要先確認一下開啟的`NodePort`資訊是什麼
```bash=
$ kubectl -n ingress-nginx get svc
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.111.134.97   <none>        80:32663/TCP,443:31309/TCP   1d
```

在我的範例環境中，預設 `HTTP` 所使用的的連接埠是 `32663`, 因此等下需要使用這個資訊來進行測試連線


首先，我們先針對 `nginx.example.com` 進行測試，結果如下圖
![Imgur](https://i.imgur.com/XxZVgHc.png)
![Imgur](https://i.imgur.com/8IL2Vh5.png)

結果如預期般，可以透過不同的 `PATH` 來導向不同的服務後端

接下來測試 `node.example.com`, 看看是否能夠針對 `host` 來導向不同的後端

![Imgur](https://i.imgur.com/CBolbM0.png)

的確也能夠正常運作，意味者我們的 `Ingress` 測試滿順利的，都能夠如預期般的運作

# Summary
本篇文章旨在透過簡單的介紹讓大家知道 `Kubernetes Ingress` 的基本架構與介紹，最後透過一個常用的 `Nginx Ingress` 實作來實際使用看看`Ingress` 的架構。
不同的 `Ingress` 的後端實現功能的方法都不盡相同，同時能夠支援的功能也都會有些許的差異，這部分就仰賴各位在選擇對應的解決方案時的研究與先行測試。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
