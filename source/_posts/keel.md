---
title: '[DevOps] 基於 Kubernetes 的自動部屬流程 - Keel'
keywords: 'Kubernetes,Keel,CI,CD,Deploy'
abbrlink: 53127
date: 2019-01-12 08:21:49
tags:
  - Kubernetes
  - CI/CD
description: 本文介紹一種基於 Kubernetes 開發的 Continuous Deployment 解決方案 keel, Keel 透過部署相關應用於 Kubernetes 內並且直接針對 Container Registry 中的 Container Image 去讀取相關的資訊，同時搭配 Semantic Versioning 2.0.0 的格式來確保印象檔的新舊，並且針對新版來進行運行資源 (Pod/Deployment..etc) 的更新

---

# Preface
隨者 `Kubernetes` 的風潮崛起，愈來愈多企業/玩家/研究人員嘗試使用 `Kubernetes` 作為其應用服務的底層平台。此外隨者 `DevOps` 的意識以及工作效率改善的風氣的流行，現在 `CI/CD` 這個詞更是眾人朗朗上口。
然而基於 `Container Image` 為軟體版本的 `Kubernetes` 平台遇到 `CI/CD(Continuous Integration/Deployment)` 流程是，是否能夠有很好的銜接方式，或是問再銜接上可能會有什麼問題? 本文要介紹的就是其中一種用來處理 `Kubernetes` 中 `CD` 流程的解決方案. `Keel`.


# Introduction
## Problem
再正式介紹 `Keel` 之前，我們先自己思考一下，如果遇到今天下列的情況，你可能會怎麼做?
1. 開發者提交程式碼，並且與 `CI` 系統有良好整合，確保程式品質
2. 希望最新版的程式碼能夠自動的部署到 `Kubernetes` 集群，不論是 `dev/staging/production/...etc`

基於上述的環境下，其實有滿多的方法可以完成，但是不同的方式其實都牽扯到不同的條件。譬如
1. 使用的 `CI` 是哪一套，是 `Drone/Jenkins/CircleCI/Travis` ? 是自架維運還是直接採用企業雲端服務?
2. 容器倉庫放置的地點是哪裡? Docker hub ? 還是公有雲相關的服務?
3. `Kubernetes` 部署的位置在哪裡? 是放置在公有雲裡面? 還是私人集群? 公有雲裡面是採用 `kubernetes service (GKE/EKS/AKS)` 還是透過 `VM` 的方式自行架設?


不同的情境，對應到上述的問題都會有不同的解法。而本文的 `Keel` 則是用來解決上述問題的`其中一種`的解決方案
假設今天使用了 `雲端服務` 為主的 `CI` 解決系統，而此時相關的 `Kubernetes` 則是直接部署在三大公有雲裡面。

這時候整個運作流程就是
1. 程式碼提交(這邊使用 `Github` 為範例), 並且觸發相關的 `CI` 流程
2. 該流程中，我們進行了相關的功能與測試，並且建置好相關的容器映像檔(Container Image)
3. 將該容器映像檔案部署到特定的容器倉庫 (Container Registry)
4. 透過特定的方式想辦法更新 `Kubernetes` 內運行的容器

上述的流程牽扯到不同雲端服務，這意味者就牽扯到不同的權限控管。
舉例來說
1. Github 與 `CI` 相關的資訊授權(不是本文重點)
2. `CI` 系統與容器倉庫平台的權限控管
    - 通常來說可能只需要 `Push Image` 相關的權限.
    - `Push Image` 可能意味 `Write` 配上一些 `Read`. 這部分沒有唯一，看平台而定
3. `CI` 系統與 `Kubernetes` 平台的權限控管
    - 這部分應該會基於 `Kubernetes` 的 `Role (RBAC)` 來決定，根據應用程式的使用方式，可能會需要更強大的權限來部署相關的資源，如 `Pod/Deployment/Service/Ingress/Secret/ConfigMap`

所以對我來說最困難的反而不是這些如何串聯起來，反而是就授權方面，該如何處理才是合宜的。

我認為權限控管沒有一定的答案，根據自己所在環境的政策以及需求，並且針對風險與開發流程去評估最適合自己的處理方式才對。

## Keel
基於上述的流程，如果今天`不希望`在 `CI` 系統擁有太多關於 `Kubernetes` 相關的權限，即不希望 `CI` 系統能夠主動的去更新 `kubernetes` 內的資源狀態。
一種相對應的解法就是，有沒有辦法讓 `Kubernetes` 自己去更新相關的資源狀態? 相關的概念可以是
1. `CI` 建置並且更新最新的 `Container Image` 後想辦法通知 `Kubernetes` 內部
2. `CI` 建置並且更新最新的 `Container Image` 後, `kubernetes` 本身自己去偵測是否需要更新相關的資源

而 `keel` 就是上述概念的解決方案，透過相關的設定，自動偵測是否有新的 `Container Image` 並且更新相關的 `kubernetes` 資源

Keel 的[官方網站](https://keel.sh/v1/guide/#What-is-Keel) 上面有更多的介紹，這邊我就直接使用其架構圖來介紹 `Keel`

以下的架構圖是基於 `Keel V1` 的版本。
![Imgur](https://i.imgur.com/naZJoMX.png)

在整個 `Keel` 的架構中，我們可以分成三大部分來看待

# Keel
## Registry
在 `Keel` 裡面會需要針對 `Container Image` 去偵測，判別是否有新的版本，因此必須要與 `Container Registry` 有所連動，本身要有相關的權限可以去讀取相關的 `Container Image Information` 來判斷是否有新版.

因此該架構圖左半部分描述的就是相關的 `Container Registry`. 從公有雲服務到自架的服務都有支援，譬如 `Dockerhub, Quay, GCR`. 此外 `程式碼比文件新`，實際上連 `AWS ECR` 也有支援.
詳細的支援列表可以參閱[Keel Trigger](https://keel.sh/v1/guide/documentation.html#Triggers)

### Detection Approach
`Keel` 要怎麼去偵測是否有新的 `Container Image`. 目前 `Keel` 裡面實現了兩種概念，分別為`被動接收 Webhook` 以及`主動定時詢問(Polling))`

`Webhook` 的概念就是到各個 `Container Registry` 去設定的 `webhook`，並且將他指向 `keel`. 當該 `Contaienr Registry` 收到對應的 `Push Image Event` 時會主動通知 `Keel`  有新版的 `Image` 被更新了.

`Polling` 則相對簡單，`Keel` 對定期的去檢視相關的 `Container Registry/Container Image` 是否有產生新版。 相對於 `Trigger` 來說, `Polling` 的反應時間會比較長.

此外，這兩個比較像是 `Webhook` 是一定會採用，而 `Polling` 是可以決定要不要使用，因此可以同時使用 `webhook + polling` 來監控，或是只有單純 `webhook`.

### Rules
由於是根據 `Container Image` 的更新來判別是否有新版, 因此在比對的規則上就特別的重要。
到底什麼叫做新版? 什麼叫做舊版? 如果每次的 `image tag` 都只是一些看似亂碼的 `git commit hash tag` 的話，其實 `keel` 根本搞不清楚誰是新版本，誰是舊版本.

因此 `Keel` 嚴格遵守 [Semantic Versioning 2.0.0](https://semver.org/), 基於 `${Major}.${Minor}.${Patch}-${Labels}` 的規範來比對
詳細的比對規則可以參閱 [Semantic Versioning 2.0.0](https://semver.org/) 來學習更多。

此外，有部分的需求是希望使用 `latest` 這種不會改變名稱的 `tag` 來進行更新。因此 `keel` 也有針對這類型的需求提供了設定方式。
針對特定不變的 `tag` 名稱, `keel` 會去讀去該 `image digest` 去判別其產生的時間，根據創建時間來判別版本的大小。


## Configuration Monitor
前述設定好相關的 `Container Registry` 之後, `Keel` 要怎麼知道去監控 `Kubernetes` 裡面的何種資源?

這部分總共提供兩種方式, 分別是 `Native Yaml` 以及 `Helm Chart`

基本上兩者的使用概念完全一樣，只是設定的方式有所不同

### Native Yaml
在 `Native Yaml` 中，由於 `Yaml` 會被 `Kubernetes API Server` 先行處理，因此在格式上不能有太多的變化，因此所有的設定都是基於 `Annotation` 以及 `Labels` 的方式來設定

譬如下列範例是透過 `keel.sh/policy: major` 去設定 `karolisr/webhook-demo:0.0.8` 的版本更新。
如果有收到任何 `webhook(預設是接收 webhook 更新)` 並且告知 `karolish/webhook-demo` 的 `image version` 有 `Major` 的版本更新，就幫我更新該運行的 `StatefulSet`
```yaml=
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wd
  namespace: default
  labels:
      name: "wd"
      keel.sh/policy: major
spec:
...
    spec:
      containers:
        - image: karolisr/webhook-demo:0.0.8
          imagePullPolicy: Always
...
```

下列範例則是採用 `Polling` 的方式，並且定期每`10分鐘`去檢查一下目標的 `Container Image` 是否有更新.

此外 `policy:force` 代表的意思就是不考慮任何版本(SemVer), 而是針對 `Image Digest` 內的創建時間來更新新版即可，因此使用 `Latest` 的時候就會使用此作法。
```yaml=
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wd
  namespace: default
  labels:
      name: "wd"
      keel.sh/policy: force
      keel.sh/trigger: poll
  annotations:
      keel.sh/pollSchedule: "@every 10m"
spec:
...
```

### Helm Chart
`Helm Chart` 的部分相對簡單,針對 `values.yaml` 去進行撰寫相關設定即可。

以下述範例來說，會透過 `Polling` 的方式每`兩分鐘` 就去嘗試看看是否有 `karolisr/webhook-demo` 的版本更新了.
而特別注意的是 `policy:all` 代表的完整的 `SemVer` 有任何新版的就會直接採用。

另外，版本的部分要透過 `keel.images.tag` 以及 `keel.image.repository` 的方式來設定，此範例則是參考到上面的 `image.repository` 以及 `image.tag`.
```yaml=
replicaCount: 1
image:
  repository: karolisr/webhook-demo
  tag: "0.0.8"
  pullPolicy: IfNotPresent
service:
  name: webhookdemo
  type: ClusterIP
  externalPort: 8090
  internalPort: 8090

keel:
  # keel policy (all/major/minor/patch/force)
  policy: all
  # trigger type, defaults to events such as pubsub, webhooks
  trigger: poll
  # polling schedule
  pollSchedule: "@every 2m"
  # images to track and update
  images:
    - repository: image.repository
      tag: image.tag
```


## Notification
假如一切都更新完成後，最後的部分就是通知了，如何將相關的部屬結果通知給管理者。
這方面 `keel` 整合了一些常見的溝通工具，譬如 `slack`, `mattermost`, `hipchat`.

詳細的整合方式請參閱 [Keel Notificaion](https://keel.sh/v1/guide/documentation.html#Notifications)

## Misc
除了上述的部分外，還有一些功能可以使用，譬如 `Approval` 等投票機制，確認同意才會部署等相關的行為。
這些都可以在 [Keel Documentation](https://keel.sh/v1/guide/documentation.html) 找到。

# Example
接下來會直接採用一個簡單的範例來測試 `keel` 的功能。

## Install Keel
安裝的部分可以採用 `Helm` 或是直接部署相關的 `Yaml` 即可。
基本上會在 `Kubernetes` 內安裝相對應的 `Deployment` 來提供上述所描述的所有功能。

```bash=
helm repo add keel-charts https://charts.keel.sh
helm repo update
helm upgrade --install keel --namespace=kube-system keel-charts/keel
```

詳細的安裝流程可參閱 [Keel Installation](https://keel.sh/v1/guide/installation.html)

安裝完畢後執行下列指令確認安裝完成
```bash=
hwchiu~$ kubectl --namespace=kube-system get pods -l "app=keel"
NAME                   READY   STATUS    RESTARTS   AGE
keel-8b8447549-wsfqr   1/1     Running   0          5s
```

## Deploy Application
這邊我是採用自己的 `Container Image` 來測試部署，並且打算採用`Dockhub` 作為後端的 `Container Registry`。 同時採用 `Polling` 的方式來偵測 `Container Image` 是否有更新

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keel-demo
  namespace: default
  labels:
      name: "keel-demo"
      keel.sh/policy: all
      keel.sh/trigger: poll
  annotations:
      keel.sh/pollSchedule: "@every 1m"
spec:
  selector:
    matchLabels:
      name: keel-demo
  template:
    metadata:
      labels:
        name: keel-demo
    spec:
      containers:
      - name: keel-demo
        image: hwchiu/netutils:0.1.0
        imagePullPolicy: Always
        name: keel-demo
```

執行下列指令確認當前運行的 `Image`.
```bash=
hwchiu:~$ kubectl get pods -l"name=keel-demo" -o jsonpath="{.items[0].spec.containers[0].image}"
hwchiu/netutils:0.1.0
hwchiu:~$
```
## Update Image
接下來我們透過 `docker tag/docker push` 的方式來更新相關的
 `docker image`.

```bash=
hwchiu:~$ sudo docker tag hwchiu/netutils:0.1.0 hwchiu/netutils:0.1.1
hwchiu:~$ sudo docker push hwchiu/netutils:0.1.1
The push refers to a repository [docker.io/hwchiu/netutils]
ab42d9bbb598: Layer already exists
98d902303c2d: Layer already exists
bcff331e13e3: Layer already exists
2166dba7c95b: Layer already exists
5e95929b2798: Layer already exists
c2af38e6b250: Layer already exists
0a42ee6ceccb: Layer already exists
0.1.1: digest: sha256:f1a3643b8b10c98b4aa9e4ac8269b7587c1d9f415f134ff359f920b8539a6f76 size: 1776
```

## CheckResources
接下來我們透過 `watch` 指令來定時觀看
```bash=
watch kubectl get pods -l"name=keel-demo" -o jsonpath="{.items[0].spec.containers[0].image}"
```

會得到下列的結果
```bash=
Every 2.0s: kubectl get pods -lname=keel-demo -o jsonpath={.items[0].spec.containers[0].image}          Sat Jan 12 07:55:42 2019

hwchiu/netutils:0.1.1
```

可以觀察到版本真的改變了，接下來嘗試修正 `Major` 版本看看
```bash=
hwchiu:~$ sudo docker tag hwchiu/netutils:0.1.1 hwchiu/netutils:1.1.1
hwchiu:~$ sudo docker push hwchiu/netutils:1.1.1
The push refers to a repository [docker.io/hwchiu/netutils]
ab42d9bbb598: Layer already exists
98d902303c2d: Layer already exists
bcff331e13e3: Layer already exists
2166dba7c95b: Layer already exists
5e95929b2798: Layer already exists
c2af38e6b250: Layer already exists
0a42ee6ceccb: Layer already exists
sleep 601.1.1: digest: sha256:f1a3643b8b10c98b4aa9e4ac8269b7587c1d9f415f134ff359f920b8539a6f76 size: 1776
hwchiu:~$ sleep 60;^C
hwchiu:~$ kubectl get pods -l"name=keel-demo" -o jsonpath="{.items[0].spec.containers[0].image}"
hwchiu/netutils:0.1.1

hwchiu:~$ sleep 60
hwchiu:~$ kubectl get pods -l"name=keel-demo" -o jsonpath="{.items[0].spec.containers[0].image}"
hwchiu/netutils:1.1.1
```

# Summary
這次跟大家介紹一款針對 `Kubernetes` 量身打造的 `CD` 工具，不過適不適合各位的環境並不是一個絕對的答案，就讓各位自己去評估是否有這個需求。 畢竟通常越方便使用，有時候其彈性反而愈少，當未來有任何客製化需求的時候可能反而會綁手綁腳。


# Reference
- [Keel](https://keel.sh/v1/guide/#What-is-Keel)
- [Semantic Versioning 2.0.0](https://semver.org/)

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