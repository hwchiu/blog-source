---
title: 11個保護你 Kubernetes 集群的技巧與觀念(下)
keywords: 'kubernetes,security,Linux,k8s,TLS'
date: 2018-07-29 17:51:49
tags:
	- kubernetes
	- security
	- linux
description:
---

本篇文章的原文為 [11 Ways Not to Get Hacked](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/#11-run-a-service-mesh)

本文作者將這 11 個技巧與觀念分成三大部分來
分別是
1. The Control Plane
2. Workloads
3. The Future


本篇文章延續 [11個保護你 Kubernetes 集群的技巧與觀念(上)](http://hwchiu.com/k8s-security-11tips-i.html#more)

繼續探討原文作者後半部分的概念

<!--more-->

在上篇文章中，我們專注於 `The Control Plane` 這邊相關的安全管理，而本篇文章我們則會專注於後續的兩個方向，分別是 `Workloads` 以及 `The Future`。


## Workloads
相對於 `The Control Plane` 著重於整個集群架構上的安全問題， `Workloads` 則是著重於運行於 `kubernetes` 內的各種工作服務，如 `Pods`,`Deployments`, `Jobs` 等
雖然這些工作服務在部署階段是受到 `kubernetes` 經過檢查確認合法才會往下執行的，但是由於這些工作服務都會直接面向使用者，若這些工作容器內本身的權限過高且遭受到非預期的攻擊時，就會衍生出其他的安全性問題。

因此作者認為，針對這些容器相關的部署，其權限能夠愈低愈好，針對其用途開啟特定權限，不能為了方便偷懶就將全部的權限開啟。


### Use Linux Security Features and PodSecurityPolicies
針對 `Linuux` 相關的容器來說，`Linux Kernel` 其實提供了不少安全控管的機制，譬如
1. Linux Capability
2. AppArmor
3. SELinux
4. seccomp

#### Linux Capability
[Linux Capability](http://man7.org/linux/man-pages/man7/capabilities.7.html) 是針對 Thread 為單位的權限控管，部分跟系統底層的操作都需要特定的 `capability` 才能夠操作，譬如擁有 `NET_ADMIN` 能力的則可以透過相關指令去修改系統上的網路設定(Routing/Interface), 而擁有 `NET_RAW` 的則有能力可以去聽取 `Raw Socket` 的封包。
在 `Kubernetes` 內也可以透拓 `SecurityContext` 對每個 `Container` 去進行相關的權限設定.
類似用法如下
```yml
pods/security/security-context-4.yaml  
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]

```

#### AppArmor
[AppArmor](https://kubernetes.io/docs/tutorials/clusters/apparmor/) 則是一種已 `Process` 程序為單位去限制其存取檔案及操作的一種機制。
針對每個 `Process` 可以去描述只能存取哪些檔案，藉此降低該程序被攻擊時所造成的傷害.

對於 `Kubernetes`來說，自從 `1.4.0` 開始就有支援 `AppArmor` 相關的功能，可以事先在每個節點上安裝 `AppArmor` 並且透過 `Profile` 的方式去設定 `AppArmor` 的權限，然後每個 `Pod` 透過 `Annotation` 的方式去掛載特定的 `AppArmor` 來限制該容器的權限。
詳細的範例可以參考官網的範例[Kubernetes AppArmor](https://kubernetes.io/docs/tutorials/clusters/apparmor/)
```yml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    # Tell Kubernetes to apply the AppArmor profile "k8s-apparmor-example-deny-write".
    # Note that this is ignored if the Kubernetes node is not running version 1.4 or greater.
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-apparmor-example-deny-write
spec:
  containers:
  - name: hello
    image: busybox
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

#### SELinux
Security-Enhanced Linux (SELinux) 主旨在於提供更強安全性的 Linux 系統，藉由取代原先的 `自主式存取控制 (Discretionary Access Control, DAC)` 為 `委任式存取控制 (Mandatory Access Control, MAC) 的方法` 來針對特定的應用程式與特定的使用者達成更細部的權限管理，可達成即使是 `root` 身份在執行特定檔案/指令時也會被降權成非`root`權限。


對應於 `Kubernetes` 來說，首先要先在節點上面開啟 `SELinux` 相關的功能，並且正確設定完畢。
接下來對於要採用該 `SELinux` 的 `Pod` 中去設定 `SecurityContext`，並且透過 `seLinuxOptions` 的方式來設定相關的參數
整體的參數結構可以參考 [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#selinuxoptions-v1-core)而使用範例可參考如下

```yml
...
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```
#### Seccomp
seccomp-bpf 是一套以 `process` 程序為單位去限制其能夠呼叫 `Linux Kernel System Call` 的機制。
一般的應用程式大部分可能都只會用到 `Read`,`Write` 等基本功能，再來可能會牽扯到 `Fork/Clone` 這類型，更甚至會牽扯到網路相關的網卡綁定。

因此透過 `seccomp` 的功能去限制能夠執行的 `System Call` 也能夠大幅的降低當應用程式被攻擊時所造成的傷害。

對應於 `kubernetes` 來說，該功能目前還是屬於 `Alpha` 階段，目前的使用方式大概是
1. 在對應的節點上放置相關的 `seccomp` 的檔案，用來描述相關的設定
2. 在運行的 `Pod` 的檔案內，透過 `Annotation` 的方式將該 `Pod` 與對應的 `seccomp` 設定檔連結來達到限制的功用。
詳細的操作可以參考[Kubernetes Pod-Security-Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy)

除了上述提到跟 `Linux Kernel` 提供的安全機制外，`Kubernetes` v1.11 本身也提供了一個新的機制 `Pod Securirt Policy`(Beta).

該機制提供了一種`Cluster-Wide`的安全性設定，能夠將相關的安全性設定設定成一份全域的設定檔案，透過 `RBAC` 以及 `ServiceAccount` 的綁定，就可以讓所有的 `Pod` 自動套用這些安全性設定，減少了每個單獨設定的繁瑣程序。
相像的文件可以參考[Kubernetes Pod-Security-Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy)

### Statically Analyse YAML

相對於前面提到這種`RunTime`時期的安全性檢查，這邊要介紹的則是一種相輔相成的概念，`Statiscally Analyse YAML`，針對部署服務的 `Yaml` 設定檔案去分析，評估該 `Yaml` 內的設定是否足夠安全。

作者認為敏感機密的資訊不應該被放在任何的 `Yaml` 內容之中，若採用 `ConfigMap` 以及 `Secret` 等方式來存放任何機密資訊的話，也必須要將這些資料透過一些工具來進行加密，譬如 [Valut(with CoreOS's operator](https://github.com/coreos/vault-operator), [git-crypt](https://github.com/AGWA/git-crypt), [sealed secrets](https://github.com/bitnami-labs/sealed-secrets) 以及 [cloud procider KMS](https://cloud.google.com/kms/)

此外，這邊也提出了另外一套我覺得滿有趣的工具 [kubectl-kubesec](https://github.com/stefanprodan/kubectl-kubesec)
該工具是一個 kubelet 的 plugin, 能夠針對部署資源的 Yaml 檔案將行掃描，根據安全性相關的設定給予評分，讓管理者能夠知道盡可能地將安全性相關的權限給濃縮到最小，減少被惡意攻擊時能夠造成的傷害。
舉例來說針對 `kubernetes-dashboard` 的部署，得到下列結果。

```
kubernetes-dashboard kubesec.io score 7
-----------------
Advise
1. containers[] .securityContext .runAsNonRoot == true
Force the running image to run as a non-root user to ensure least privilege
2. containers[] .securityContext .capabilities .drop
Reducing kernel capabilities available to a container limits its attack surface
3. containers[] .securityContext .readOnlyRootFilesystem == true
An immutable root filesystem can prevent malicious binaries being added to PATH and increase attack cost
4. containers[] .securityContext .runAsUser > 10000
Run as a high-UID user to avoid conflicts with the host's user table
5. containers[] .securityContext .capabilities .drop | index("ALL")
Drop all capabilities and add only those required to reduce syscall attack surface
```
其建議大概如下

1. 不要用 `Root`去運行，應該修改 Container 改成 `non-root` 就可以執行
2. Linux Capabilityes 應該要移除掉沒有用到的部分
3. 針對 `/` Root File System 應該只能夠唯獨就好
4. 執行 User 的 UID 應該要超過 10000，可以避免該 UID 與本機端的 Host 衝突(主要是因為 User Namespace 還沒有完全實作於 `k8s` container 中)


### Run Containers as a Non-Root User
作者說道，大部分使用 `Root` 去運行的 `Container` 應用程式其實根本都不需要 `Root` 這麼大的權限，以 `Root` 身份去執行擁有的權限已經超過的該應用程式真正需要的權限，這樣的設定會讓任何惡意的攻擊者在成功突入後有更大的權限去對系統造成更大的傷害。

首先，目前的 Linux 在針對檔案權限的部分預設還是採用 `自主式存取控制 (Discretionary Access Control, DAC)` 這種依據檔案的 `User/Group UID/GID` 對應的方式來判別是否有權限進行讀寫。

同時，在 `Kubernetes` 的環境下，創建各式各樣的 `Container` 的同時並沒有啟動`User namespace` 的隔離，這意味者容器內使用者的ID (UID) 實際上是可以對應到外面節點上的使用者資料集中。

以實際的案例來說明上述的情況就是若以 `Root` 的身份去運行一個容器於 `kubernetes` 集群之中，在對應的工作節點上可以觀察到實際上也是用一個`Root`的身份去運行該容器的應用程式。
以下面的範例來說，我們在 `kubernetes` 節點上可以觀察到 `coredns` 這個 `pod` 實際上是用 `root` 身份在節點上去運行對應的應用程式。

```bash
$ps axuw |grep dns
vagrant   4656  0.0  0.0  12916   936 pts/3    S+   07:38   0:00 grep --color=auto dns
root      7886  0.2  0.5  45224 21924 ?        Ssl  Jul27   5:56 /coredns -conf /etc/coredns/Corefile
root      8011  0.2  0.5  45224 22180 ?        Ssl  Jul27   6:04 /coredns -conf /etc/coredns/Corefile
```

雖然目前已經有很多的技術被提出來避免 `Container` 內的應用程式可以突破限制進而影響外部宿主機上面的資源，但是作者認為能夠避免使用 `Root` 去執行還是避免使用，畢竟安全性這種東西沒有完美的一天。

此外，大部分的容器應用程式都會使用 `Root` 的身份執行第一個應用程式，也就是 `PID=1` 的應用程序。如果該應用程式被攻擊者掌控了也就意味者攻擊者獲得了 `Root`的權限，因此攻擊者就能夠執行更進一步的操作。


文中特別提及一篇文章 [Running Non-Root Containers On Openshift](https://engineering.bitnami.com/articles/running-non-root-containers-on-openshift.html), 說到 `Bitnami` 將他們用到的 Container 從 `Root` 轉移到 `Non-Root` 的過程以及方法，並且說明為什麼要轉移到 `Non-Root` 的執行環境。

最後，作者在此提到了 `kubernetes` v1.11 的新功能 (PodSecuurityPolicy),
使用了一個簡單的 `Yaml`  來描述如何限制所有的 Pod 都必須要以 `Non-Root` 的身份來執行
``` yaml
# Required to prevent escalations to root.
allowPrivilegeEscalation: false
runAsUser:
  # Require the container to run without root privileges.
  rule: 'MustRunAsNonRoot'
```

最後作者提到，到 `User Namespace` 隔離的功能完成之前，使用 `Non-Root` 身份去運行容器依然是一個必要且不可以避免的選擇，為了整個集群以及容器的安全性，還是需要多花時間進行轉換。

### Use Network Policies
講完了跟容器部署的安全性之後，接下來要探討的是`Pod`之間互相存取的安全性問題
在預設的情況下，`kubernetes` 集群內的`Pod`彼此都可以互相透過網路存取，即使是不同 `namespace` 的容易也可以。

如果想要限制`Pod`間的網路存取，這邊可以引入 `kubernetes` 提供的功能 `Network Policy`, 透過 `Network Policy`, 管理者可以限制`Pod`間的存取規則，除了單純的 `Pod` 之外，還可以限制連接埠,`Namespace` 以及相關的網路協定 (TCP/UDP).

但是因為 `Network Policy` 實際上並不是 `kubernetes` 自行實作的內容，這部分要仰賴 `kubernetes` 集群使用的 CNI(Container Network Interface).
如果想要嚐鮮的人可以考慮使用看看 `Calico`, `Weave` 等有支援 `Network Policy` 的 `CNI`。


就如同之前所提供的概念一樣，針對安全性的部分都是盡可能的縮小權限，非必要的就不要開啟。
套用在 `Network Policy` 這邊則是非必要的連線都不允許通過，所以一開始可以先講所有的連線全部都阻擋起來，再根據需要的連線一條一條的規則打開。
這樣的做法聽起來有效，然而卻非常的複雜與繁瑣，因此這邊又介紹另外一個有趣的工具
[netassert](https://github.com/controlplaneio/netassert),一套基於 `nmap`的工具，可以根據設定去測試當前的封包走向是否運作，藉此減少撰寫 `NetworkPolicy` 時要不停的打流量來測試當前的 `Network Policy`是否如設定般運作。

### Scan Images and Run IDS
最後一個要探討的安全部分則是針對 `Image` 本身進行安全性的探討。
在 `kubernetes` 的流程中，當 `Pod` 本身的 `yaml` 送進到 `kubernetes` 集群中後，會受到 `Admission Controllers`  一系列的檢查，去確保這次資源的部屬是合法的。
這中間的檢查，有一個非常有趣的就是 `webhook`. 透過 `webhook`, 我們可以透過第三方的應用程式去檢查當前的 `Image` 安全與否，若不安全就阻擋下來，讓該 `Pod` 沒有機會被部署到 `kubernetes` 集群內。

關於 `webhook` 的使用方法，可以參考官方文件的說明 [kuubernetes webhook](https://kubernetes.io/docs/reference/access-authn-authz/webhook/
).

至於檢查 `Image` 這個行為，我們暫時稱作 IDS(Intrusion Detection System), 這邊作者提供了兩套相關的工具來提供這類型的服務，分別是 [Clair](https://github.com/coreos/clair) 以及 [Micro Scanner](https://github.com/aquasecurity/microscanner).

除了上述這種工具外，作者也提到另外一個服務的概念, [grafeas](https://grafeas.io/), 該服務會維護一個資料庫，該資料庫內會以每個 Image 的 Hash tag 作為索引資料，來記住對應的 Image 是否有任何安全性的問題。
所以可以藉由此工具來掃描當前所屬的 Image 是否內部繼承了任何有問題的 Image 來判斷是否有任何安全性問題。

至於 `Zero Day` 這種類型的安全性問題也有相關的工具可以幫忙處理，如[twistlock](https://www.twistlock.com/),[aquasec](https://www.aquasec.com/) 以及 [Sysdig Secure](https://sysdig.com/product/secure/)

## The Future
最後，在學習了 `Control-Plane` 以及 `Workload` 這種兩類型相關的安全性防護後，作者認為接下來的趨勢則是 `Service Mesh`.

`Service Mesh` 簡單來說就是在不修改所有已經存在的微服務容器前提下，透過一些第三方的應用程式讓你在不同容器中間建立一條加密的連線，將這些服務串起來。

此外這些容器的連線中也有所謂的 `Zero Trust`連線，藉由 `TLS` 相關的加密處理安全性問題，減少了需要 `Network Policy` 額外的封包處理。

基本上現在最火紅的 `Service Mesh` 服務就是 `Istio`，這部分的概念與用法非常龐大，短短一篇文章沒有辦法訴說完畢，之後若有時間可以再好好的寫一篇文章來介紹 `Istio 的概念與用法`。


## Summary
總算將這一篇安全性相關的文章給唸完了，裡面的內容其實很短，但是這 11 個重點其實每一個都可以獨立寫成一篇文章來仔細探討。

作者將整個安全性分成三個部分，總共 11 點來探討，這 11 點除了介紹概念之外，也都提供了不少相關 kubernetes 的用法或是額外的工具來協助各式各樣的安全性問題。

最後，我認為安全性真的很重要，如果今天你要提供一個`kubernetes`平台供外部使用者使用的話，在最後進行到 `Production` 前一定要花時間注意當前集群內的安全性問題，存 `Control-Plane` 先確保基礎架設的安全，接下來從 `Workloads` 方面去限制相關資源部署的權限問題，使用最小的權限來執行所需要的應用程式，降低問題發生時的傷害。


## Reference
- [11 Ways (Not) to Get Hacked](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked)
