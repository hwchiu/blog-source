---
title: '[Kubernetes] DNS Setting with Dockerd(原始碼分析下)'
keywords: 'kubernetes,DNS,Linux,k8s'
abbrlink: 65056
date: 2018-08-15 07:38:53
tags:
  - Kubernetes
  - Linux
  - DNS
  - SourceCode
  - Docker
description:
---

此篇文章是 Kubernetes Pod-DNS 系列文章最後一篇
此系列文會從使用者的用法到一些問題的發掘，最後透過閱讀程式碼的方式去分析這些問題

相關的文章連結如下
- [[Kubernetes] DNS setting in your Pod](https://www.hwchiu.com/kubernetes-dns.html)
- [[Kubernetes] DNS Setting with Dockerd](https://www.hwchiu.com/kubernetes-dns-ii.html)
- [[Kubernetes] DNS Setting with Dockerd(原始碼分析上)](https://www.hwchiu.com/kubernetes-dns-iii.html)

## 正文

再上篇文章 - [[Kubernetes] DNS Setting with Dockerd(原始碼分析上)](https://www.hwchiu.com/kubernetes-dns-iii.html) 中，我們透過閱讀 `kubernetes` 原始碼的方式已經理解到 `kubernetes` 本身是怎麼去處理 `DNS` 的設定。 但是卻留下了一個謎團就是 `dockerd` 到底如何處理每個容器本身的 `DNS` 設定?

重新複習一次之前觀察到的結果


| node\dockerd | 有設定 DNS | 沒設定 DNS |
| -------- | -------- | -------- |
| 有數值     | node     | node     |
| 沒有數值     | dockerd     | 8.8.8.8     |	


這邊直接開門見山的說明結論
當每次創建容器時，大抵上都會遵循下列邏輯
1. 先根據參數`resolveConf` 來讀取當前 `DNS` 的全部設定
2. 如果使用者有自行設定 `DNS` 的參數，就會全面使用這邊的設定，完全忽略(1)載入的設定
3. 如果使用者沒有自行設定 `DNS` 的話，就會針對 (1) 載入的設定進行一次過濾
    - 針對 127.0.0.1/8 之類的進行過濾
    - 過濾後若發現是空的，則補上 8.8.8.8/8.8.4.4


接下來我們就會透過直接閱讀 `docker-ce` 原始碼的方式來理解這個問題

<!--more-->

## dockerd

如同前篇文章所說，要透過閱讀原始碼找尋問題的最大困難點就是起點，要先想辦法找到與目標問題有關的進入點，然後從該進入點開始挖掘跟目標有關的程式碼。

這邊使用上次的那張流程圖，將其修改一下來符合本篇的方向來看。

首先，當 `kubelet` 透過 `CRI (Contaienr Runtime Interface)` 將容器相關的操作請求送到 `docekrd` 後就會開始進行容器的創見/刪除/修改

假設我們要進行的創建容器這個操作，以 `dockerd` 的角度來說用比較簡單的方式來看大概就是
1. 創建容器
2. 對該容器設定相關規則
    - 網路相關功能
    - AppArmor 相關
    - ...等各式各樣的功能
 
![Imgur](https://i.imgur.com/HBxFvzA.png)


上面只是一個比較抽象的方式去描述整個過程，我們想要瞭解的`DNS`相關的設定都在所謂的 `Config Network` 裡面。

接下來幫大家節省時間，直接將詳細用到的程式碼流程列在下列圖表中
![Imgur](https://i.imgur.com/AElxokT.png)

我們接下來就直接從整個創造容器的起點 `ContaienrStart` 來開始看吧!

## 程式碼研究

### ContainerStart
```go ContaienrStart https://github.com/docker/docker-ce/blob/da1e08d48493406ce290a1b99269e52879af5b0e/components/engine/daemon/start.go#L18 start.go

func (daemon *Daemon) ContainerStart(name string, hostConfig *containertypes.HostConfig, checkpoint string, checkpointDir string) error {
...
	// check if hostConfig is in line with the current system settings.
	// It may happen cgroups are umounted or the like.
	if _, err = daemon.verifyContainerSettings(container.OS, container.HostConfig, nil, false); err != nil {
		return errdefs.InvalidParameter(err)
	}
	// Adapt for old containers in case we have updates in this function and
	// old containers never have chance to call the new function in create stage.
	if hostConfig != nil {
		if err := daemon.adaptContainerSettings(container.HostConfig, false); err != nil {
			return errdefs.InvalidParameter(err)
		}
	}
	return daemon.containerStart(container, checkpoint, checkpointDir, true)
}
```

這邊的邏輯基本上就是處理一些 `Container` 相關設定，最後直接呼叫一個私有函式 `containerStart` 來進行更進一步的處理

### containerStart



```go containerStart https://github.com/docker/docker-ce/blob/da1e08d48493406ce290a1b99269e52879af5b0e/components/engine/daemon/start.go#L102 start.go

// containerStart prepares the container to run by setting up everything the
// container needs, such as storage and networking, as well as links
// between containers. The container is left waiting for a signal to
// begin running.
func (daemon *Daemon) containerStart(container *container.Container, checkpoint string, checkpointDir string, resetRestartManager bool) (err error) {

....
	if err := daemon.conditionalMountOnStart(container); err != nil {
		return err
	}

	if err := daemon.initializeNetworking(container); err != nil {
		return err
	}

....
	if daemon.saveApparmorConfig(container); err != nil {
		return err
	}
...
	err = daemon.containerd.Create(context.Background(), container.ID, spec, createOptions)
	if err != nil {
		return translateContainerdStartErr(container.Path, container.SetExitCode, err)
	}

	// TODO(mlaventure): we need to specify checkpoint options here
	pid, err := daemon.containerd.Start(context.Background(), container.ID, checkpointDir,
		container.StreamConfig.Stdin() != nil || container.Config.Tty,
		container.InitializeStdio)
...

	container.SetRunning(pid, true)
	container.HasBeenManuallyStopped = false
	container.HasBeenStartedBefore = true
	daemon.setStateCounter(container)

	daemon.initHealthMonitor(container)
...    
}
```

這個函式非常的重要，可以看一下該函式的註解
{% note danger %}
// containerStart prepares the container to run by setting up everything the
// container needs, such as storage and networking, as well as links
// between containers. The container is left waiting for a signal to
// begin running.
{% endnote %}

再這個函式內會創建好相關的容器，並且會將該容器用到的相關資源(儲存/網路)等都準備好
，由於我們要觀察的是 `DNS` 相關的資訊，所以我們要繼續往 `initializeNetworking` 的方向往下追。

### initializeNetworking

```go initializeNetworking https://github.com/docker/docker-ce/blob/6e92e5909666b3b9c2aecebf582e8af85f228899/components/engine/daemon/container_operations.go#L916 container_operations.go

func (daemon *Daemon) initializeNetworking(container *container.Container) error {
	var err error

	if container.HostConfig.NetworkMode.IsContainer() {
...
	}

	if container.HostConfig.NetworkMode.IsHost() {
...
    }

	if err := daemon.allocateNetwork(container); err != nil {
		return err
	}

	return container.BuildHostnameFile()
}

```

接下來都是根據各種 `Networking` 相關的設定來處理，前面兩個的判斷處理則是根據當初創建該容器時，有沒有指令 `--network=xxxx` 來特別處理，若有設定 `--network=host` 或是 `--network=container:xxxx` 會有一些額外的處理。

因為這些情境跟我們的 `Kubernetes` 的使用方法不同，我們不會走到這邊的判斷，而是直接會走到下面的 `allocateNetwork` 來開始準備網路相關的資訊。

### allocateNetwork

```go allocateNetwork https://github.com/docker/docker-ce/blob/6e92e5909666b3b9c2aecebf582e8af85f228899/components/engine/daemon/container_operations.go#L501 container_operations.go
func (daemon *Daemon) allocateNetwork(container *container.Container) error {
...

	// always connect default network first since only default
	// network mode support link and we need do some setting
	// on sandbox initialize for link, but the sandbox only be initialized
	// on first network connecting.
	defaultNetName := runconfig.DefaultDaemonNetworkMode().NetworkName()
	if nConf, ok := container.NetworkSettings.Networks[defaultNetName]; ok {
		cleanOperationalData(nConf)
		if err := daemon.connectToNetwork(container, defaultNetName, nConf.EndpointSettings, updateSettings); err != nil {
			return err
		}

	}
...
	return nil
}
```
再這個函式內，首先會嘗試使用預設的網路型態，這邊指的就是在創建容器所下的參數 `--net=xxx`，而預設的類型就是 `bridge`.
所以接下來就很直覺的去呼叫 `connectToNetwork` 來進行下一階段的處理


### connectToNetwork 
```go connectToNetwork https://github.com/docker/docker-ce/blob/6e92e5909666b3b9c2aecebf582e8af85f228899/components/engine/daemon/container_operations.go#L690 container_operations.go
func (daemon *Daemon) connectToNetwork(container *container.Container, idOrName string, endpointConfig *networktypes.EndpointSettings, updateSettings bool) (err error) {
	start := time.Now()
	if container.HostConfig.NetworkMode.IsContainer() {
		return runconfig.ErrConflictSharedNetwork
	}
	if containertypes.NetworkMode(idOrName).IsBridge() &&
		daemon.configStore.DisableBridge {
		container.Config.NetworkDisabled = true
		return nil
	}

....
	sb := daemon.getNetworkSandbox(container)
	createOptions, err := buildCreateEndpointOptions(container, n, endpointConfig, sb, daemon.configStore.DNS)
	if err != nil {
		return err
	}
....
	if sb == nil {
		options, err := daemon.buildSandboxOptions(container)
		if err != nil {
			return err
		}
		sb, err = controller.NewSandbox(container.ID, options...)
		if err != nil {
			return err
		}

		updateSandboxNetworkSettings(container, sb)
	}
....
	return nil
}
```
`connectToNetwork` 並不是只有創建新的容器時才會使用，所以這邊還會進行一些相關的參數檢查。
由於我們是第一次串件該容器，所以容器所對應的沙盒 `SandBox` 會是空的。
最後根據空的沙盒，決定透過 `controller.NewSanbox` 去創建一個沙盒。
值得注意的是，這邊有一個 `buildSandboxOptions` 會把其他用到的參數都重新整理一次，然後傳入到 `controller.NewSandbox` 這邊去處理。


### NewSandbox
```go    NewSandbox https://github.com/docker/libnetwork/blob/f30a35b091cc2a431ef9856c75c343f75bb5f2e2/controller.go#L1126 controller.go
func (c *controller) NewSandbox(containerID string, options ...SandboxOption) (Sandbox, error) {
	if containerID == "" {
		return nil, types.BadRequestErrorf("invalid container ID")
	}

	var sb *sandbox
...

	// Create sandbox and process options first. Key generation depends on an option
	if sb == nil {
		sb = &sandbox{
			id:                 sandboxID,
			containerID:        containerID,
			endpoints:          []*endpoint{},
			epPriority:         map[string]int{},
			populatedEndpoints: map[string]struct{}{},
			config:             containerConfig{},
			controller:         c,
			extDNS:             []extDNSEntry{},
		}
	}

	sb.processOptions(options...)
...
    if err = sb.setupResolutionFiles(); err != nil {
		return nil, err
	}

	
	return sb, nil
}
```
{% note danger %}
特別注意一下，當我們到這邊執行相關函式的時候，我們的專案已經從`docker-ce`遷移到`libnetwork`了。
{% endnote %}

這個函式非常的長，描述的創建新的 `Sandbox` 期間需要注意的所有事項，這邊首先會透過`sb.processOptions` 去設定相關變數的數值，這邊要特別注意的是其實每一個 `options` 本身是對應到一個 `function pointer`。

{% note danger %}
如果我們當初有透過 `dockerd` 去設定相關的 `DNS` 設定的話，這邊其實實際上會賦值到 `sb.config.dnsList`, `sb.config.dnsSearchList` 以及 `sb.config.dnsOptionsList`
{% endnote %}


```go setupResolutionFiles https://github.com/docker/libnetwork/blob/c3a682c10b554b2ff2fac8ca134ddb9047ffdd93/sandbox_dns_unix.go#L61 sandbox_dns_unix.go
func (sb *sandbox) setupResolutionFiles() error {
	if err := sb.buildHostsFile(); err != nil {
		return err
	}

	if err := sb.updateParentHosts(); err != nil {
		return err
	}

	return sb.setupDNS()
}
```
這邊其實非常簡單，我們的主要目標終於出現了!!!
`setupDNS` 意思就如同名稱一樣直接，就是設定該沙盒內`DNS`的設定。


### setupDNS
```go setupDNS https://github.com/docker/libnetwork/blob/c3a682c10b554b2ff2fac8ca134ddb9047ffdd93/sandbox_dns_unix.go#L181 sandbox_dns_unix.go
func (sb *sandbox) setupDNS() error {
	var newRC *resolvconf.File
....

	originResolvConfPath := sb.config.originResolvConfPath
	if originResolvConfPath == "" {
		// if not specified fallback to default /etc/resolv.conf
		originResolvConfPath = resolvconf.DefaultResolvConf
	}
	currRC, err := resolvconf.GetSpecific(originResolvConfPath)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
		// it's ok to continue if /etc/resolv.conf doesn't exist, default resolvers (Google's Public DNS)
		// will be used
		currRC = &resolvconf.File{}
		logrus.Infof("/etc/resolv.conf does not exist")
	}

	if len(sb.config.dnsList) > 0 || len(sb.config.dnsSearchList) > 0 || len(sb.config.dnsOptionsList) > 0 {
		var (
			err            error
			dnsList        = resolvconf.GetNameservers(currRC.Content, types.IP)
			dnsSearchList  = resolvconf.GetSearchDomains(currRC.Content)
			dnsOptionsList = resolvconf.GetOptions(currRC.Content)
		)
		if len(sb.config.dnsList) > 0 {
			dnsList = sb.config.dnsList
		}
		if len(sb.config.dnsSearchList) > 0 {
			dnsSearchList = sb.config.dnsSearchList
		}
		if len(sb.config.dnsOptionsList) > 0 {
			dnsOptionsList = sb.config.dnsOptionsList
		}
		newRC, err = resolvconf.Build(sb.config.resolvConfPath, dnsList, dnsSearchList, dnsOptionsList)
		if err != nil {
			return err
		}
		// After building the resolv.conf from the user config save the
		// external resolvers in the sandbox. Note that --dns 127.0.0.x
		// config refers to the loopback in the container namespace
		sb.setExternalResolvers(newRC.Content, types.IPv4, false)
	} else {
		// If the host resolv.conf file has 127.0.0.x container should
		// use the host resolver for queries. This is supported by the
		// docker embedded DNS server. Hence save the external resolvers
		// before filtering it out.
		sb.setExternalResolvers(currRC.Content, types.IPv4, true)

		// Replace any localhost/127.* (at this point we have no info about ipv6, pass it as true)
		if newRC, err = resolvconf.FilterResolvDNS(currRC.Content, true); err != nil {
			return err
		}
		// No contention on container resolv.conf file at sandbox creation
		if err := ioutil.WriteFile(sb.config.resolvConfPath, newRC.Content, filePerm); err != nil {
			return types.InternalErrorf("failed to write unhaltered resolv.conf file content when setting up dns for sandbox %s: %v", sb.ID(), err)
		}
	}

	// Write hash
	if err := ioutil.WriteFile(sb.config.resolvConfHashFile, []byte(newRC.Hash), filePerm); err != nil {
		return types.InternalErrorf("failed to write resolv.conf hash file when setting up dns for sandbox %s: %v", sb.ID(), err)
	}

	return nil
}
```
這個函式會針對一些跟 `DNS` 相關的參數來進行處理，包含了
1. dnsServer
2. dnsSearch
3. dnsOptions
4. resolveConf

這邊的運作邏輯如下
1. 先根據參數`resolveConf`來讀取當前 `DNS` 的全部設定
2. 如果使用者有自行設定 `DNS` 的參數，就會全面使用這邊的設定，完全忽略(1)載入的設定
2.1 這邊最後會呼叫 `resolvconf.Build` 將參數的設定直接覆寫到容器內的 `/etc/resolv.conf`
3. 如果使用者沒有自行設定 `DNS` 的話，就會針對 (1) 載入的設定進行一次過濾
3.1 針對 127.0.0.1/8 之類的進行過濾 `FilterResolveDNS`
3.2 過濾後若發現是空的，則補上 8.8.8.8/8.8.4.4


這樣就來看一下 `FilterResolveDNS` 怎麼處理 `DNS`

### FilterResolveDNS
``` FilterResolveDNS https://github.com/docker/libnetwork/blob/c3a682c10b554b2ff2fac8ca134ddb9047ffdd93/resolvconf/resolvconf.go resolvconf.go

// FilterResolvDNS cleans up the config in resolvConf.  It has two main jobs:
// 1. It looks for localhost (127.*|::1) entries in the provided
//    resolv.conf, removing local nameserver entries, and, if the resulting
//    cleaned config has no defined nameservers left, adds default DNS entries
// 2. Given the caller provides the enable/disable state of IPv6, the filter
//    code will remove all IPv6 nameservers if it is not enabled for containers
//
func FilterResolvDNS(resolvConf []byte, ipv6Enabled bool) (*File, error) {
	cleanedResolvConf := localhostNSRegexp.ReplaceAll(resolvConf, []byte{})
	// if IPv6 is not enabled, also clean out any IPv6 address nameserver
	if !ipv6Enabled {
		cleanedResolvConf = nsIPv6Regexp.ReplaceAll(cleanedResolvConf, []byte{})
	}
	// if the resulting resolvConf has no more nameservers defined, add appropriate
	// default DNS servers for IPv4 and (optionally) IPv6
	if len(GetNameservers(cleanedResolvConf, types.IP)) == 0 {
		logrus.Infof("No non-localhost DNS nameservers are left in resolv.conf. Using default external servers: %v", defaultIPv4Dns)
		dns := defaultIPv4Dns
		if ipv6Enabled {
			logrus.Infof("IPv6 enabled; Adding default IPv6 external servers: %v", defaultIPv6Dns)
			dns = append(dns, defaultIPv6Dns...)
		}
		cleanedResolvConf = append(cleanedResolvConf, []byte("\n"+strings.Join(dns, "\n"))...)
	}
	hash, err := ioutils.HashData(bytes.NewReader(cleanedResolvConf))
	if err != nil {
		return nil, err
	}
	return &File{Content: cleanedResolvConf, Hash: hash}, nil
}

```
- 首先先呼叫 `ReplaceAll` 把所有 `localhost 127.0.0.0/8` 相關的 `IP` 都清空。
- 清空之後，若發現這時候沒有 `DNS` 的話，直接透過 `dns := defaultIPv4Dns` 補上預設的 `DNS` (8.8.8.8/8.8.4.4)


## summary
當每次創建新容器時，最後會依賴到 `libnetwork` 內跟 `DNS` 相關的參數來設定
1. 如果使用者有自行設定 `DNS` 的參數，就會全面使用這邊的設定，完全忽略(1)載入的設定
    - 這邊最後會呼叫 `resolvconf.Build` 將參數的設定直接覆寫到容器內的 `/etc/resolv.conf`
2. 如果使用者沒有自行設定 `DNS` 的話，就會針對 (1) 載入的設定進行一次過濾
    - 針對 127.0.0.1/8 之類的進行過濾 `FilterResolveDNS`
    - 過濾後若發現是空的，則補上 8.8.8.8/8.8.4.4
