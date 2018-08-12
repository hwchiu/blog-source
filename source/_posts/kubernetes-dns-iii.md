---
title: '[Kubernetes] DNS Setting with Dockerd(原始碼分析上)'
keywords: 'kubernetes,DNS,Linux,k8s'
abbrlink: 24894
date: 2018-08-12 16:02:59
tags:
  - Kubernetes
  - Linux
  - DNS
  - SourceCode
description:
---

在前篇文章
[[Kubernetes] DNS Setting with Dockerd](https://www.hwchiu.com/kubernetes-dns-ii.html) 中已經詳細介紹了整個問題的流程以及觀察結果。

再次重申一次結論
`kubernetes` 會先嘗試使用節點上 `/etc/resolv.conf` 的資料，但是若發現 `/etc/resolv.conf` 是空的，這時候就會去依賴 `dockerd` 幫忙產生的 `/etc/resolv.conf`


本篇會直接從 `kubernetes` 以及 `docker` 的原始碼來研究這個問題，並且佐證上篇文章的觀察結果。

<!--more-->>

## Kubernetes
在觀察這個現象前，我們必須要先思考方向，該怎麼去追這段程式碼?
到底程式碼的開頭應該從哪邊開始追起?

要講起這個問題實在不是一言兩語可以解決的，這部份除非本身已經對 `kubernetes`原始碼很瞭解，否則就是要倚賴關鍵字加上本身對程式語言的經驗來判斷，趕緊找到一個正確的進入點。

這邊直接先使用一個簡單的流程圖，來描述整個運作的邏輯，然後接下來會針對這段流程進行更進一步的分析來解決我們的疑問

## CRI 流程圖

這邊我們先簡單的知道， `kubernetes` 本身有非常多的插件，包含了 `Storage`, `Network`, `Device` 以及 `Runtime`。
這就是你會常常聽到 `CNI`, `CRI`, `CSI` 以及 `Device Plugin` 等這些名詞出現的理由

`Kubernetes` 藉由這些插件把部分功能都給模組化，讓任何符合該標準的第三方套件都能夠與 `kubernetes` 緊密合作。
> 未來將會撰寫一系列的文章來介紹 CNI，從 CNI 概念到自己撰寫一個 CNI，敬請期待

因為今天的問題主要是運行容器內的設定問題，這部份則是會跟 `CRI(Contaienr Runtime Interface)` 有關連, 因此我透過下圖稍微說明一下整個流程。

在每個 `kubernetes` 的節點上都會運行一隻守護程序 `kubelet`，該 `kubelet` 內部有一個稱為 `kube-runtime` 的套件會用來管理在該節點上所有跟容器有關的操作。

`kube-runtime` 本身會遵守 `CRI` 的規範來管理容器，而 `CRI` 則是建立在 `gRPC` Client/Server 的架構下運行的。

因此 `kube runtime` 本身會扮演者 `gRPC` client 的角色，對 gRPC server `dockershim` 發送管理容器的請求。

而 `dockershim` 則是一個 `kubernetes` 內部自行設計實作的一個基於 `docker container` 且相容 `CRI` 的 gRPC server.

當 `dockershim` 收到請求後，就會透過 `Docker Container` 本身的 `API` 進行 `Docker Container` 相關的資源處理。

![Imgur](https://i.imgur.com/t5jQGmd.png)

## 程式碼追蹤

藉由上述流程圖的幫忙後，我們可以有更進一步的方向去思考程式的進入點。

首先我們知道透過對 `Pod` 設定 `dnsPolicy:default` 才會有這個問題，同時也觀察到 `dockerd` 本身的設定也會影響整個 `/etc/resolve.conf` 的結果。

所以 `kube runtime` 這邊是一個可能線索，如何解析 `Pod` 本身的參數並處理是一個方向
同時 `dockershim` 這邊如何針對 `docker container` 則會是另外一個線索來追尋。

針對這兩個方向，經過仔細的追尋後，我們可以得到類似下圖的流程
> 圖中藍色區域都是真實的函式名稱

![Imgur](https://i.imgur.com/6xsRAD1.png)

當 **1** 號發出`Create Pod`的請求出來後， `kubelet` 裡面於 `kube runtime` 模組內則會呼叫到 `createPodSandbox` 的函式。
在 `createPodSandbox` 則會進行下列事情
1. 呼叫 `generatePodSandboxConfig`
2. 透過 `CRI` 要求遠方執行 `RunPodSandbox`

### kube runtime
#### createPodSandbox
整個函式如下
```go=
// createPodSandbox creates a pod sandbox and returns (podSandBoxID, message, error).
func (m *kubeGenericRuntimeManager) createPodSandbox(pod *v1.Pod, attempt uint32) (string, string, error) {
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
	if err != nil {
		message := fmt.Sprintf("GeneratePodSandboxConfig for pod %q failed: %v", format.Pod(pod), err)
		glog.Error(message)
		return "", message, err
	}

	// Create pod logs directory
	err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)
	if err != nil {
		message := fmt.Sprintf("Create pod log directory for pod %q failed: %v", format.Pod(pod), err)
		glog.Errorf(message)
		return "", message, err
	}

	podSandBoxID, err := m.runtimeService.RunPodSandbox(podSandboxConfig)
	if err != nil {
		message := fmt.Sprintf("CreatePodSandbox for pod %q failed: %v", format.Pod(pod), err)
		glog.Error(message)
		return "", message, err
	}

	return podSandBoxID, "", nil
}
```

#### generatePodSandboxConfig
`generatePodSandboxConfig` 這個函式很簡單，基本上就是從 `Pod` 本身 `yaml` 檔案裡面去讀取設定，根據這些設定來處理相關的資訊。
我們在意的部份就是 `DNS` 相關，所以以下就擷取 `DNS` 相關的函式內容

```go=
func (m *kubeGenericRuntimeManager) generatePodSandboxConfig(pod *v1.Pod, attempt uint32) (*runtimeapi.PodSandboxConfig, error) {
....
	dnsConfig, err := m.runtimeHelper.GetPodDNS(pod)
	if err != nil {
		return nil, err
	}
	podSandboxConfig.DnsConfig = dnsConfig
.....
```

這邊可以看到呼叫了 `GetPodDNS` 來取得對應的設定，然後我們就一路往下追

#### GetPodDNS

```go=
func (c *Configurer) GetPodDNS(pod *v1.Pod) (*runtimeapi.DNSConfig, error) {
	dnsConfig, err := c.getHostDNSConfig(pod)
	if err != nil {
		return nil, err
	}

	dnsType, err := getPodDNSType(pod)
	if err != nil {
		glog.Errorf("Failed to get DNS type for pod %q: %v. Falling back to DNSClusterFirst policy.", format.Pod(pod), err)
		dnsType = podDNSCluster
	}
	switch dnsType {
	case podDNSNone:
		// DNSNone should use empty DNS settings as the base.
		dnsConfig = &runtimeapi.DNSConfig{}
	case podDNSCluster:
        ...
	case podDNSHost:
		// When the kubelet --resolv-conf flag is set to the empty string, use
		// DNS settings that override the docker default (which is to use
		// /etc/resolv.conf) and effectively disable DNS lookups. According to
		// the bind documentation, the behavior of the DNS client library when
		// "nameservers" are not specified is to "use the nameserver on the
		// local machine". A nameserver setting of localhost is equivalent to
		// this documented behavior.
		if c.ResolverConfig == "" {
			switch {
			case c.nodeIP == nil || c.nodeIP.To4() != nil:
				dnsConfig.Servers = []string{"127.0.0.1"}
			case c.nodeIP.To16() != nil:
				dnsConfig.Servers = []string{"::1"}
			}
			dnsConfig.Searches = []string{"."}
		}
	}
...
    return c.formDNSConfigFitsLimits(dnsConfig, pod), nil
}
```

這邊的邏輯非常簡單
1. 先透過 `getHostDNSConfig` 取得當前 `/etc/resolv.conf` 的內容，並且存於變數 **dnsCOnfig**
2. 接下來透過 `getPodDNSType` 該函式從 `Pod Yaml` 內讀取對應的 `dnsPolicy` 資訊，來決定當前 `Pod` 的 `dnsType`
3. 第三步驟就是最重要的，根據當前 `dnsType` 來決定要如何處理 `dnsConfig`.
由於 `podDNSCluster` 的程式碼太多了，我這邊就先忽略掉，我們只要專注看 `DNSHost` 的案例。
裡面非常簡單，如果你當初運行 `kubelet` 的時候有特別設定不要使用 `/etc/resolv.conf` 的話，就會把 `dnsConfig` 補上一個 `127.0.0.1` 的資訊，不過這個 `Case` 我暫時還沒想到，可能就是機器本身就是一個 `DNSServer` 的情況吧。
4. 將 `dnsConfig` 回傳出去
所以在正常情況下，我們的 `dnsConfig` 理論上就會是節點上 `/etc/resolv.conf` 內的資料。

到這邊為止，就是 `kube-runtime`  自行處理的部份，並且將得到的 `dnsConfig` 放置到變數 `podSandboxConfig.DnsConfig` 之中。

接下來我們來看一下 `dockershim` 裡面的 `RunPodSandbox` 會怎麼處理這些 `DNS` 的資訊。

### DocekrShim
#### RunPodSandbox

由於 `RunPodSandbox` 的函式非常長，這邊就擷取跟 `DNS` 相關的部份來觀察

```go=
func (ds *dockerService) RunPodSandbox(ctx context.Context, r *runtimeapi.RunPodSandboxRequest) (*runtimeapi.RunPodSandboxResponse, error) {
	config := r.GetConfig()
....
	// Rewrite resolv.conf file generated by docker.
	// NOTE: cluster dns settings aren't passed anymore to docker api in all cases,
	// not only for pods with host network: the resolver conf will be overwritten
	// after sandbox creation to override docker's behaviour. This resolv.conf
	// file is shared by all containers of the same pod, and needs to be modified
	// only once per pod.
	if dnsConfig := config.GetDnsConfig(); dnsConfig != nil {
		containerInfo, err := ds.client.InspectContainer(createResp.ID)
		if err != nil {
			return nil, fmt.Errorf("failed to inspect sandbox container for pod %q: %v", config.Metadata.Name, err)
		}

		if err := rewriteResolvFile(containerInfo.ResolvConfPath, dnsConfig.Servers, dnsConfig.Searches, dnsConfig.Options); err != nil {
			return nil, fmt.Errorf("rewrite resolv.conf failed for pod %q: %v", config.Metadata.Name, err)
		}
	}
....
	return resp, nil
}
```
其實這邊的程式碼已經有了滿不錯的註解來解釋相關的行為。
1.之前透過 `kube run-time` 得到的 `dns` 設定都只是一個存在於記憶體內的一堆資料而已，其實都還沒有跟真正的容器有任何關係。
2.透過 `dockershim` 的操作，這時候其實已經將對應的容器創見完畢，而且該容器內`/etc/resolv.conf`的內容是完全依賴 `docker container` 決定
3.`kubernetes` 這邊的邏輯非常簡單，如果我之前有產生過任何 `dns` 的設定，就直接將 `docker`  產生出來的 `/etc/resolv.conf` 給直接覆蓋掉。

所以我們可以看一下 `rewriteResolvFile` 這個函式
#### rewriteResolvFile

```go=
// rewriteResolvFile rewrites resolv.conf file generated by docker.
func rewriteResolvFile(resolvFilePath string, dns []string, dnsSearch []string, dnsOptions []string) error {
...
	var resolvFileContent []string
	for _, srv := range dns {
		resolvFileContent = append(resolvFileContent, "nameserver "+srv)
	}

	if len(dnsSearch) > 0 {
		resolvFileContent = append(resolvFileContent, "search "+strings.Join(dnsSearch, " "))
	}

	if len(dnsOptions) > 0 {
		resolvFileContent = append(resolvFileContent, "options "+strings.Join(dnsOptions, " "))
	}

	if len(resolvFileContent) > 0 {
		resolvFileContentStr := strings.Join(resolvFileContent, "\n")
		resolvFileContentStr += "\n"

		glog.V(4).Infof("Will attempt to re-write config file %s with: \n%s", resolvFilePath, resolvFileContent)
		if err := rewriteFile(resolvFilePath, resolvFileContentStr); err != nil {
			glog.Errorf("resolv.conf could not be updated: %v", err)
			return err
		}
	}

	return nil
}
```

這邊也不能理解，就是根據之前存放於 `podSandboxConfig.DnsConfig` 內各式各樣關於 `DNS` 的設定組合起來，然後直接覆蓋掉本來的 `/etc/resolv.conf`.

## 結論

到這邊，我們已經把整個問題給釐清一半了

重新看一次之前的結論

>再次重申一次結論
`kubernetes` 會先嘗試使用節點上 `/etc/resolv.conf` 的資料，但是若發現 `/etc/resolv.conf` 是空的，這時候就會去依賴 `dockerd` 幫忙產生的 `/etc/resolv.conf`

我們的推論跟我們程式碼觀察的結果是完全吻合的，再 `dnsPolicy=default` 的前提下，只要 `kubernetes` 只要能夠獲得合法的 `/etc/resolv.conf` 就會使用，否則直接使用 `docekr container` 所創造的 `/etc/resolv.conf`.

但是這邊還有留下一個謎點，到底 `dockerd` 的設定是如何影響 `/etc/resolv.conf` 以及 `8.8.8.8/8.8.4.4` 是如何出現的?

由於再寫下去本篇文章會愈來愈長，所以決定將 `docker container` 相關的程式碼分析再寫一篇文章來處理。
