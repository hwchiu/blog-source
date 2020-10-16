---
title: 使用 Travis CI 為你的 Kubernetes 應用程式打造自動化測試
keywords: 'k8s,Kubernetes,GitHub,CI/CD,Travis CI,TravisCI'
tags:
  - DevOps
  - Kubernetes
  - CI/CD
  - Ubuntu
  - Linux
abbrlink: 9763
date: 2018-07-21 08:49:11
description: 這篇文章的主軸其實非常簡單，目標是希望為開發者的 Kubernetes 應用程式提供更強健的自動化測試流程，確保該應用程式在開發上能夠與 Kubernetes 緊密結合。為了確保程式品質，我們都會在開發的過程中撰寫一些單元測試/整合測試來確保開發的功能能夠正常運作。 特別是當有新功能的開發或是臭蟲修復時不會對舊有的功能造成損毀。 這個理念看起來非常合理，但是當應用程式一旦與 Kubernetes 結合的時候，這個理念到底好不好實現?

---

作為一個 `kubernetes` 玩家以及 Backend 開發者，我觀察到縱使很多人都將 Backend Server 放到 `kubernetes` 上去運行，但是因為實際上可能是

隸屬於不同的成員去處理這些事情，透過分工的方式來協調這些工作。

雖然大部分的應用程式基本上不太在意是否有 `kubernetes` 的存在，只要確保本身的功能正常運作即可，譬如 `MongoDB` 等應用程式。

但是有些應用程式本身會對 `kubernetes` 集群進行操作，這類型的應用程式的功能都要存在一個 `kubernetes` 集群來互動，確保功能正常

而本篇文章存在的目的就是要提供一種解決方案給後者的應用程式，讓你的應用程式能夠更容易地與 `Kubernetes` 結合來進行測試。

<!--more-->


一句話講清楚用途
=============

本篇文章提供的解決方案，能夠為你的應用程式提供每一次程式碼提交測試都產生一個獨立的 `Kubernetes` 集群進行測試，這樣有下列好處
1. 不同程式碼提交使用不同集群，互相不衝突，不會有污染問題
2. 由於使用不同集群，也不用擔心其他的測試會對集群造成資源汙染，導致後續的測試失敗
3. 在最基本的要求下，這套解決方案是完全免費的，完全依賴免費的服務方案組合而成。


元件介紹
=======

在本篇文章的解決方案中，我們會使用到下列的元件來展示，當然有些元件都可以自由變化。
只要掌握整個流程的理解，原件都可以自行替換成自己習慣使用的服務。

### GitHub
放置應用程式原始碼的地方，本案例中使用 `GitHub` 這個程式碼託管網站。

### TravisCI
一套自動化測試的服務，與 `GitHub` 可以連動，當你的 `GitHub` 專案有收到任何程式碼合併請求(`Pull Request`) 等更動的時候，可以透過 `TravisCI`幫你的應用程式進行測試，並且將測試的結果回傳到 `GitHub` 讓專案管理員瞭解該次的程式碼修改是否有順利通過所有測試。

### Minikube
一套用來建置 `Kubernetres` 集群的工具，在我們的方案中我們會在 `TravisCI` 的環境中透過此工具來產生一個本地的 `kubernetes` 集群。接者我們 `TravisCI` 內的應用程式就獲得一個獨立的 `kubernetes` 集群來使用了。

本篇文章不會介紹關於 `Gitnub` 以及 `TravisCI` 的基本操作，而是著重在這兩者與 `kubernetes` 的整合。
因此相關操作請自行學習。


架構介紹
=======

接下來將使用下圖來介紹整個測試方案的運作流程。

![](https://i.imgur.com/kTmhZsb.png)
1. 假設開發者的 `GitHub` 專案已經與 `TravisCI` 進行連接
2. 開發者日以繼夜的撰寫程式碼，對 `GitHub` 專案發出程式碼合併更動的請求
3. `GitHub` 這邊收到通知，通知 `TravisCI` 準備進行測試
4. `TravisCI` 根據需求創建一個全新的測試環境出來
    - 這邊則會產生一個虛擬機器出來
5. 在上述產生的測試環境中，使用 `Minikube` 該元件來產生一個全新的 `kubernetes` 集群
6. 確認集群創建完畢，相關服務的運行中後，便可以針對 `GitHub` 專案內的程式碼進行測試
7. 測試的結果回報給 `TravisCI`，然後 `TravisCI` 會再回報給 `GitHub` 專案，讓開發者可以瞭解這次的修改是否有通過所有的測試。


示範案例展示
==========

本次示範所使用的程式碼都可以於 [kubeTravisDemo](https://github.com/hwchiu/kubeTravisDemo) 內找到

而該專案對應的 `TravisCI` 結果也可以在 [TravisCI kubeTravisDemo](https://travis-ci.org/hwchiu/kubeTravisDemo/builds) 瀏覽


## 應用程式
本篇開頭提到，有些應用程式會需要與 `Kubernetes` 有緊密的連結操作，而本次展示的專案則使用了 `Client-go` 這個套件開發了一個很簡單能夠自行產生 `Pod` 的應用程式。
並且為該功能撰寫了一個簡單的測試，該測試會透過該函式嘗試去產生 `Kubernetes` Pod 並且確保該 Pod 有成功產生，最後將其刪除。


> 如果想要了解怎麼透過 `client-go` 撰寫 `kubernetes` 相關應用程式，可參閱下列投影片 [Kubernetes library 開發實戰 with client-go](https://speakerdeck.com/chenyunchen/kubernetes-library-with-client-go)

該測試程式碼大致上如下
```golang
....
	err = createPod(clientset, podName)
	assert.NoError(t, err)

	pod, err := clientset.CoreV1().Pods("default").Get(podName, metav1.GetOptions{})
	assert.NotNil(t, pod)
	assert.NoError(t, err)

	err = clientset.CoreV1().Pods("default").Delete(podName, &metav1.DeleteOptions{})
	assert.NoError(t, err)
...
```
如果在一個沒有 `kubernetes` 集群的環境中，該測試程式碼則沒有辦法測試(因為沒有真的集群可以去進行`kubernetes`操作)

> 雖然有 `Fake-client` 可以進行相關的測試，但是有部份的操作是需要真的集群去運行才可以進行的，這類型的就沒有辦法用 `Fake-client` 來測試。
>

## TravisCI
當應用程式準備好之後，我們就要在 `GitHub` 專案中描寫我們如何使用 `TravisCI` 的測試環境。
該環境描述檔案採用 `yaml` 的格式，名稱為 `.travis.yml`，可以在 [Travis CI Getting started](https://docs.travis-ci.com/user/getting-started/) 看到相關文件.

這邊有幾件事情要注意
1. Travis-CI 產生的是虛擬機器，預設沒特別開啟情況下，我們不能在虛擬機器內再開啟另外一個虛擬機器，所以我們要使用 `Docker` 的方式來創造 `kubernetes` 集群. 這部分可以透過 `minikube --vm-drive=none` 來達成
2. 在 minikube 的部署方式中，有 `localkube` 以及 `kubeadm` 兩種方式來部署，由於目前 `minikube` 主推 `kubeadm` 並且也說明未來會拋棄 `localkube`，因此我們的部屬方式決定採用 `kubeadm`.
3. 由於 `kubeadm` 本身會依賴 `systemd` 去進行相關的 `systemd service` 運行，而 `Ubuntu 14.04` 預設依然使用 `upstart` 而非 `systemd`，因此我們必須要標明我們希望使用的 `OS` 是 `Ubuntu 16.04(xenial)`

有了這幾個基本注意事項後，我們可以撰寫一個 `.travis.yml` 來符合我們的需求

```yml
language: go

sudo: required
dist: xenial
services:
  - docker

env:
  - CHANGE_MINIKUBE_NONE_USER=true

before_install:
  - go get -u github.com/kardianos/govendor

before_script:
  - sudo mount --make-rshared /
  - curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
  - curl -Lo minikube https://github.com/kubernetes/minikube/releases/download/v0.28.1/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
  - sudo minikube -v 9 start --vm-driver=none --bootstrapper=kubeadm --kubernetes-version=v1.10.0 --extra-config=apiserver.authorization-mode=RBAC
  - minikube update-context
  - until kubectl get nodes minikube | grep " Ready";do kubectl get nodes; sleep 1; done
  - until kubectl -n kube-system get pods -lk8s-app=kube-dns -o jsonpath="{.items[0].status.phase}" | grep "Running" ;do sleep 1;echo "waiting for kube-addon-manager to be available"; kubectl get pods --all-namespaces; done

script:
  - go test -v ./...
```

上述檔案內，跟 `kubernetes` 有關的部分落在 `dist`, `env` 以及 `before_script` 這三大區塊內。

### Dist
就如同上面說明，希望 `TravisCI` 所配置的機器是使用 `Ubuntu 16.04`, 免除了 `Systemd` 相關依賴的自行安裝手續
> 事實上非常難裝且有滿多問題，沒心力不要嘗試在 14.04 內去透過 systemd/kubeadm 安裝 kubernetes

### Env
這邊特別設置一個環境變數是給 `minikube` 使用的，因為一邊透過 `kubectl` 操作 `kubernetes` 集群實際上是透過 `$HOME/.kube/config(default)` 內的設計去跟集群獲得授權來操作，那在 `minikube` 的環境中，我們透過 `CHANGE_MINIKUBE_NONE_USER=true` 這個環境變數可以讓產生出來的可以被任何使用者帳號讀取使用，否則預設是 `root:root` 才有權限存取

### Before_Script

1. 這邊先針對 `/` 這個 `Mount Point` 設定成 `rshard`，供 `kube-dns` 使用
> 想瞭解 `rshard` 可以參考 [Kubernetes Mount Propagation](https://medium.com/kokster/kubernetes-mount-propagation-5306c36a4a2d) 這篇文章，有詳細的說明 Volume 間的 `rshard/rslave` 以及後來新的 feature volume bidirectional 的介紹
2. 接下來就是下載 `kubectl` 以及 `minikube` 兩個相關的執行檔
3. 接者透過 `minikube` 來創建本地的 `kubernetes` 集群
    - `-v 9` 是顯示更多的 log，可以方便安裝期間偵錯
    - `--vm-driver=none` 採用 docker 的方式而不是 VM 的方式來創建 `kubernetes`
    - `--bootstrapper=kubeadm` 指定使用 `kubeadm` 進行 `kubernetes` 叢集的安裝
    - `--kubernetes-version=v1.10.0` 這邊則是指定 `kubernetes` 的版本
4. 最後是預設將授權的方式使用 `RBAC`, 相關的 `issue` 可以參閱 [minikube issue #1722](https://github.com/kubernetes/minikube/issues/1722)
5. 到了這一步後我們的 `kubernetes` 集群就正式啟動了，但是因為一些核心的服務，如 `kube-dns` 等可能還沒有正常運作，這邊就根據你的需求看看是否需要確認該服務已經啟動後才進行應用程式的測試
6. 後續的部分都是透過一些腳本語言的方式確認 `kubernetes` 集群已經正常運作，這部分有很多寫法，依照自己喜好即可。

### script
1. 當上述的腳本都執行完畢後，意味者集群可以開始運行，我們可以進行自己應用程式的測試
2. 該測試則會真的對剛創立的 `kubernetes` 集群進行 `Pod `的創立/查詢/刪除 等行為。

相關測試
======
針對上述使用的示範專案以及相關的 `TravisCI`, 首先我們先來看消耗時間，畢竟如果每次測試都花上太多時間，其實也是會消磨大家的耐心的

這邊直接擷取 `TravisCI` 的[測試報告](https://travis-ci.org/hwchiu/kubeTravisDemo/builds/406523654), 該測試總共花費的時間是 ` 2 min 48 sec`

![](https://i.imgur.com/x8615aH.png)

左邊顯示的是相關指令，而右邊則是顯示該指令花費的時間。
1. 花費了將近 `50` 秒在進行 go 相關 library 的安裝，這是應用程式需要的，跟 `kubernetes 無關
2. 安裝 kubectl/minikube 等相關工具，這邊花費了大概 `13` 秒左右`
3. 啟動 `minikube` 大概花費了 `53` 秒
4. 最後等待 `kubernetes`及集群中的 `kube-dsn` 啟動則花費了 `23` 秒左右.

零零總總算一下，大概總共有 `90` 秒的時間再啟動一個 `kubernetes` 的集群，目前大概是耗費了整體測試的 `90/168`=`53%`
但是這個時間是固定的，基本上當本體的應用程式測試的時間愈複雜，花費的時間愈久，這個創立集群花的時間也就微不足道了。

其他
===
Q: 我就愛用 `ubuntu-14.04` 就是沒有 `systemd` 該怎麼辦?
A: 為了生命良好，先改用 `bootstrapper=localkube`

Q: 只能用在 `TravisCI` 嗎? 其他的測試服務如 `CircleCI`,`Jenkins` 能不能這樣?
A: 基本上概念一樣，你可以用各種方式去安裝`kubernetes`集群，事實上我也有在 `Jenkins` 做過類似的事情，此外除了 `minikube` 外你可也可以用 `Vagrant + kubeadm + Jenkins` 去做到類似的事情

Q: 我的測試功能可能會要有更複雜的需求跟更複雜的環境，可以怎麼辦
A: `TravisCI` 本身虛擬出來的環境還是比較死，不夠靈活，可以改用 `Jenkins` 等自己有辦法掌握虛擬機器內容的服務

Q: 有辦法支援 `kubernetes` v1.11.0 嗎
A: 這要依賴 `minikube` 官方的支援

參考
===
1. https://blog.travis-ci.com/2017-10-26-running-kubernetes-on-travis-ci-with-minikube
2. https://github.com/travis-ci/travis-ci/issues/7260

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/D7RZGWrNK

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