---
title: Install GPU in GKE(Google Kubernetes Engine)
keywords: 'k8s,Kubernetes,GPU,KGE,Cloud'
tags:
  - GPU
  - GKE
  - Kubernetes
abbrlink: 53546
date: 2018-07-14 17:41:45
description: 過去所謂的 GPU 運算幾乎都是個人主機的市場，任何人都可以很輕鬆的購買顯卡回家並且安裝起來使用，不論是玩遊戲所需要，或是需要 GPU 運算，這部分的生態早就存在已久。 隨者 AI 市場的蓬勃發展， GPU 資源的需求量上升，個人家用所擁有的數量在部分應用情境下所需要的時間太長了，因此就有了公有雲與 GPU 結合的需求出現。使用者可以根據自身的需求去評估需要多少張 GPU 卡，然後在對應的公有雲上去申請對應的資源來使用。本文基於這種概念的情況下，跟大家分享一下在 Google Kubernetes Engine 上若要使用 GPU 的話，大致上有哪些步驟需要執行，並且分享一些背後的安裝原理。

---

# Preface
最近嘗試要再 GKE 上面開啟 GPU 的服務，然後實際上卻遇到了一些問題，因此用本篇文章來記錄遇到問題並且遇到問題的解決思路。

一開始有個需求時，很直覺的再 Google 上面直接用 `GKE GPU` 進行搜尋，的確也可以找到不少文章在講 `GPU` 與 `Kubernetes` 相關。

第一個搜尋結果則是 Google 官方的文章 [Kubernetes Engine, GPUs](https://cloud.google.com/kubernetes-engine/docs/concepts/gpus)

直接整理該文章中的重點流程.
1. 從 GCP 的管理面板中去取得 GPU 資源
2. 創造 GPU Node Pool 供 GKE ㄐ
3. 透過 `kubernetes DaemonSet` 的方式安裝 `NVIDIA Drivers` 到所有 GPU Node 上
4. 接下來創造 Pod 即可正常使用 `NVIDIA` GPU 資源

# Install

前面兩個步驟基本上不會有什麼問題，透過網頁介面點選或是參考網頁中的指令去創造即可順利完成。
比較困擾的則是第三個步驟，根據該篇文章顯示我們只需要部署一個 `yaml` 檔案即可完全 `NVIDIA Drives` 到所有 GPU 節點上

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/stable/nvidia-driver-installer/cos/DaemonSet-preloaded.yaml
```

但是我在另外一個 `kubernetes` 的文章中 [Kubernetes scheduling-gpus](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
若要再 `GKE` 上面安裝 GPU 資源的話，則需要根據情況部署下列 `yaml` 檔案。

```
On your 1.9 cluster, you can use the following commands to install the NVIDIA drivers and device plugin:

# Install NVIDIA drivers on Container-Optimized OS:
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/DaemonSet.yaml

# Install NVIDIA drivers on Ubuntu (experimental):
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/nvidia-driver-installer/ubuntu/DaemonSet.yaml

# Install the device plugin:
kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.9/cluster/addons/device-plugins/nvidia-gpu/DaemonSet.yaml
```

首先，第一篇文章表明只需要透過 `DaemonSet` 的方式去安裝 `NVIDIA Drivers`，而第二篇文章則需要部署兩個 `DaemonSet`，分別來處理 `NVIDIA Drivers` 以及 `Device Plugin`.

此外，針對第一個 `NVIDIA Drivers` 也提供不同作業系統的檔案(cos/ubuntu)，且就我的認知上 `Device Plugin` 本來就需要跑一個 `DaemonSet` 來監聽 Kubernetes 的事件來處理各式各樣的 Device 操作。

在百思不得其解的情況下，乾脆直接找到相關的 Git 專案來看看，有沒有更清楚的解釋，畢竟我們都知道文件更新的速度比不上程式碼修改的速度。

最後找到了這個 Git 專案 [GoogleCloudPlatform/container-engine-accelerators](https://github.com/GoogleCloudPlatform/container-engine-accelerators/tree/master/cmd/nvidia_gpu)

這個專案的文件直接解決了我關於 `Device Plugin` 的疑惑


>In GKE, from 1.9 onwards, this DaemonSet is automatically deployed as an addon. Note that DaemonSet pods are only scheduled on
nodes with accelerators attached, they are not scheduled on nodes that don't have any accelerators attached.

根據這份文件敘述，在 GKE (1.9版本之後)，會自動部署 `Daemonset` 到集群上面的 GPU Node 來處理 `Device Plugin` 相關的事宜。

由於我使用的 `GKE` 集群版本是 1.9 之後，所以只需要自己手動部署 `DaemonSet` 來安裝 `NVIDIA Drivers` 到所有的 GPU 節點上。
結合了上述兩者個文件，可以根據版本以及GPU 節點上面的作業系統選擇需要使用的 `Yaml`.
在 Github 的專案內，使用了 `Brnach` 的方式來切換不同的 `kubernetes` 版本，所以這邊就根據各自的需求來選擇。
想要獲得如官方範例的 `raw.githubusercontent.com` 的連結只要到該 Github 的檔案中點選右上的 `Raw` 按鈕則會獲得一個關於該檔案的存取連結。
譬如
```
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/DaemonSet.yaml
```

由於該 `yaml` 使用的 `namespace` 是在 `kube-system`, 所以若要觀察其結果別忘記切換對應的 `namespace` 來觀察。

這邊只要該 `Pod`的狀態變成 `Running` 則代表該 `NVIDIA Driver` 已經安裝完畢了，就可以開始部署自己的 Pod 來使用這些 GPU 資源了。

---------------------------------
題外話:
為什麼安裝一個 `NVIDIA Driver` 完成會是 `Running` 而不是其他的 `Completed` 等看起來結束的資訊?

這邊主要受限於目前 `Kubernetes` 資源的使用方法與 `NVIDIA Driver` 的安裝使用情境。

首先，我們希望 `NVIDIA Driver` 能夠安裝到每一台擁有 GPU 的節點上，所以這邊我們會希望使用 `DaemonSet` 來幫你部署對應的 Pod 到 GPU 節點。

然而 `NVIDIA Driver` 的安裝其實本身就是一個會結束的應用程式，如果正常使用上，當相關資源都安裝完畢，該 Pod 則會結束 並且將狀態切換成 `Succeed`.

此時 `DaemonSet` 則會為了確保其選擇的 `Pod` 是一個類似 `Daemon` 的程式，必須一直存活，則又會重新將該 Pod 叫起來繼續重新安裝。

如此則會發生一個反覆執行的無窮迴圈，為了解決這個問題，不少人在提出有沒有類似 `DaemonJob` 的資源可以使用?
希望可以在每個符合的節點上都運行一個`Job`就好，不需要一直反覆執行，
可以在下列的連結看到相關的討論串.

1.[CronJob DaemonSet (previously ScheduledJob)](https://github.com/kubernetes/kubernetes/issues/36601)
2.[[DaemonSet] Considering run-once DaemonSet (Job DaemonSet)](https://github.com/kubernetes/kubernetes/issues/50689)
3.[Run Once DaemonSet on Kubernetes](http://blog.phymata.com/2017/05/13/run-once-DaemonSet-on-kubernetes/)

在這個問題被正式解決前，有不少 `Workarouund` 可以處理這個情境，最常用的採用 `Init-Container` 的方式來完成。
藉由 `Init-Container` 來執行主要的安裝流程，然後再 `Contianer` 裡面使用則是只用了一個低資源沒做事情單純無窮迴圈卡住的應用程序。

當 `Pod` 開始運行的時候，先執行 `Init-Contaienr` 來進行真正的任務來安裝相關資源，接下來切換到 `Contaienr` 進行一個發呆的應用程式
確保 `Pod` 的狀態會維持在 `Running` 而不會被 `Daemonset` 給重啟。
這也是為什麼我們只要看到 `Running` 就可以視為 `NVIDIA Driver` 已經安裝完畢了。
另外，其實 `Flannel` (CNI) 也是使用這種方式來安裝相關的 CNI 資源(config/bianry) 到每台機器上。

---------------------------------

回到正題，按照第一篇文章提到的說明文件，我們簡單部署一個 `Pod` 的應用程式來使用 GPU 資源

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-gpu-container
    resources:
      limits:
       nvidia.com/gpu: 1
```

看似簡單合理結果結果最後運行起來遇到問題，Pod 內的 GPU 應用程式在運行時會運行錯誤，錯誤訊息類似
```
"libcuda.so.1 file" is not found.
```
這看起來就是找不到 `CUDA` 相關的函式庫所以沒有辦法正常的啟用 GPU 應用程式。

這時候再重新檢視了一下該份 Google 文件，有個段落在講述 `CUDA` 的函式庫

```
CUDA® is NVIDIA's parallel computing platform and programming model for GPUs. The NVIDIA device drivers you install in your cluster include the CUDA libraries.

CUDA libraries and debug utilities are made available inside the container at /usr/local/nvidia/lib64 and /usr/local/nvidia/bin, respectively.

CUDA applications running in Pods consuming NVIDIA GPUs need to dynamically discover CUDA libraries. This requires including /usr/local/nvidia/lib64 in the LD_LIBRARY_PATH environment variable.
```

首先，你安裝 `NVIDIA Driver` 的同時， 也會一起安裝 `CUDA` 的函式庫
再來 CUDA 相關的函式庫與使用工具都可以在 `Container` 內的 `/usr/local/nvidia/lib64` 以及 `/usr/local/nvidia/bin` 找到與使用。
最後你必須要設定你應用程式的 `LD_LIBRARY_PATH` 來確保你的應用程式在尋找連結庫的時候不要忘記去找 `/usr/local/nvidia/lib64`.

有了上述這三個敘述，我重新檢視了一次自己運行的 GPU 應用程式。
首先, `LD_LIBRARY_PATH` 有正確的設定，但是我在系統上卻看不到任何 `/usr/local/nvidia/` 任何檔案，連整個資料夾都沒有，也完全找不到任何 `libcuda.so.1` 的檔案。

所以我開始懷疑 `NVIDIA Driver` 到底有沒有正確的安裝成功?

我們重新看一次該安裝的 `yaml` 檔案，如下

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-driver-installer
  namespace: kube-system
  labels:
    k8s-app: nvidia-driver-installer
spec:
  selector:
    matchLabels:
      k8s-app: nvidia-driver-installer
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: nvidia-driver-installer
        k8s-app: nvidia-driver-installer
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-accelerator
                operator: Exists
      tolerations:
      - key: "nvidia.com/gpu"
        effect: "NoSchedule"
        operator: "Exists"
      volumes:
      - name: dev
        hostPath:
          path: /dev
      - name: nvidia-install-dir-host
        hostPath:
          path:
      - name: root-mount
        hostPath:
          path: /
      initContainers:
      - image: gcr.io/google-containers/ubuntu-nvidia-driver-installer@sha256:eea7309dc4fa4a5c9d716157e74b90826e0a853aa26c7219db4710ddcd1ad8bc
        name: nvidia-driver-installer
        resources:
          requests:
            cpu: 0.15
        securityContext:
          privileged: true
        env:
          - name: NVIDIA_INSTALL_DIR_HOST
            value: /home/kubernetes/bin/nvidia
          - name: NVIDIA_INSTALL_DIR_CONTAINER
            value: /usr/local/nvidia
          - name: ROOT_MOUNT_DIR
            value: /root
        volumeMounts:
        - name: nvidia-install-dir-host
          mountPath: /usr/local/nvidia
        - name: dev
          mountPath: /dev
        - name: root-mount
          mountPath: /root
      containers:
      - image: "gcr.io/google-containers/pause:2.0"
        name: pause
```

我們可以觀察到下列幾個重點
1. Init-Container 的名稱是 `nvidia-driver-installer`
2. 該 `DaemonSet` 將節點上的 `/home/kubernetes/bin/nvidia` 透過 `HostPath` 方式掛載到 `Init-Container` 內的 `/usr/local/nvidia`.

接下來我們使用下列指令去觀察該 `Init-Container nvidia-driver-installer` 的安裝訊息。
```bash
kubectl logs -n kube-system -l name=nvidia-driver-installer -c nvidia-driver-installer
```

其輸出的資訊如下，而且 `nvidia-smi` 的資訊的確也有抓到 `Tesla K80` 這張卡，看起來一切都正常。

```
+ COS_DOWNLOAD_GCS=https://storage.googleapis.com/cos-tools
+ COS_KERNEL_SRC_GIT=https://chromium.googlesource.com/chromiumos/third_party/kernel
+ COS_KERNEL_SRC_ARCHIVE=kernel-src.tar.gz
+ TOOLCHAIN_URL_FILENAME=toolchain_url
+ CHROMIUMOS_SDK_GCS=https://storage.googleapis.com/chromiumos-sdk
+ ROOT_OS_RELEASE=/root/etc/os-release
+ KERNEL_SRC_DIR=/build/usr/src/linux
+ NVIDIA_DRIVER_VERSION=384.111
+ NVIDIA_DRIVER_DOWNLOAD_URL_DEFAULT=https://us.download.nvidia.com/tesla/384.111/NVIDIA-Linux-x86_64-384.111.run
+ NVIDIA_DRIVER_DOWNLOAD_URL=https://us.download.nvidia.com/tesla/384.111/NVIDIA-Linux-x86_64-384.111.run
+ NVIDIA_INSTALL_DIR_HOST=/home/kubernetes/bin/nvidia
+ NVIDIA_INSTALL_DIR_CONTAINER=/usr/local/nvidia
++ basename https://us.download.nvidia.com/tesla/384.111/NVIDIA-Linux-x86_64-384.111.run
+ NVIDIA_INSTALLER_RUNFILE=NVIDIA-Linux-x86_64-384.111.run
+ ROOT_MOUNT_DIR=/root
+ CACHE_FILE=/usr/local/nvidia/.cache
+ set +x
[INFO    2018-07-14 14:57:09 UTC] Running on COS build id 10323.85.0
[INFO    2018-07-14 14:57:09 UTC] Checking if third party kernel modules can be installed
[INFO    2018-07-14 14:57:09 UTC] Checking cached version
[INFO    2018-07-14 14:57:09 UTC] Found existing driver installation for image version 10323.85.0           and driver version 384.111.
[INFO    2018-07-14 14:57:09 UTC] Configuring cached driver installation
[INFO    2018-07-14 14:57:09 UTC] Updating container's ld cache
[INFO    2018-07-14 14:57:14 UTC] Verifying Nvidia installation
Sat Jul 14 14:57:15 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.111                Driver Version: 384.111                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K80           Off  | 00000000:00:04.0 Off |                    0 |
| N/A   31C    P0    62W / 149W |      0MiB / 11439MiB |    100%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
[INFO    2018-07-14 14:57:16 UTC] Found cached version, NOT building the drivers.
[INFO    2018-07-14 14:57:16 UTC] Updating host's ld cache
```

不過至少我們知道了，該 `nvidia-driver-installer` 會將相關的資源都安裝到 `/usr/local/nvidia` 而該路徑則會對應到該 GPU 節點上面的 `/home/kubernetes/bin/nvidia` 資料夾。

這樣想了一下，我覺得我們的應用程式在使用的時候，應該要手動的將相關資源掛載進來，所以整個 `yaml` 檔案修改如下


```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-gpu-container
    resources:
      limits:
       nvidia.com/gpu: 1
    volumeMounts:
    - mountPath: /usr/local/bin/nvidia
      name: nvidia-debug-tools
    - mountPath: /usr/local/nvidia/lib64
      name: nvidia-libraries
  volumes:
  - hostPath:
      path: /home/kubernetes/bin/nvidia/bin
      type: ""
    name: nvidia-debug-tools
  - hostPath:
      path: /home/kubernetes/bin/nvidia/lib64
      type: ""
    name: nvidia-libraries
```

我們手動將 GPU 節點上面的檔案掛載到容器內，其對應的路徑如下

`/home/kubernetes/bin/nvidia/bin`  ->  `/usr/local/nvidia/lib64`
`/home/kubernetes/bin/nvidia/lib64` ->  `/usr/local/bin/nvidia`

接下來到我們的應用程式內就可以正確地找到了相關的檔案，如 `libcuda.so.1` 以及相關的測試工具。
這時候執行一下 `nvidia-smi` 卻發現不能執行，會直接得到下列的錯誤
>"Failed to initialize NVML: Unknown Error"

嘗試 Google 這類型的錯誤都沒有辦法找到答案來處理這個問題，最後決定還是自己來研究一下哪裡出錯了
這邊使用的工具是 `strace`，能夠顯示出該應用程式運行過程中呼叫到的所有 `system call`，是個非常強大好用的除錯工具。

跑了一下 `strace` 馬上就發現事情的不對勁，由該輸出結果可以看到類似下列的錯誤訊息(原諒我沒有完整 log)
>read(/dev/nvidiactl) no permission

這邊可以看到 `nvidia-smi`想要嘗試讀取 `/dev/nvidiactl` 結果卻沒有權限，這怎像都覺得一定是容器權限不夠，所以補上 `Privileged` 試試看。
就發現一切都順利了, `nvidia-smi` 也不會噴錯，而應用程式也能夠順利運行了。

附上最後的 `yaml` 檔案如下

```
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-gpu-container
    resources:
      limits:
       nvidia.com/gpu: 1
    volumeMounts:
    - mountPath: /usr/local/bin/nvidia
      name: nvidia-debug-tools
    - mountPath: /usr/local/nvidia/lib64
      name: nvidia-libraries
    securityContext:
      privileged: true
  volumes:
  - hostPath:
      path: /home/kubernetes/bin/nvidia/bin
      type: ""
    name: nvidia-debug-tools
  - hostPath:
      path: /home/kubernetes/bin/nvidia/lib64
      type: ""
    name: nvidia-libraries
```

# Summary

總結一下上述的所有歷程
1. 如果今天 GKE 的版本是 1.9 之後，我們只需要運行一個 `DaemonSet` 去安裝 `NVIDIA Driver` 即可， `Device Plugin` 會自己被運行
2. CUDA 相關的資源都需要自己從 GPU 節點上掛載到所有運行的 `Pod` 中
3. 記得設定 `privileged=true` 到所有使用 GPU 的節點上。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/mLwKb3zrx

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4XNG

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