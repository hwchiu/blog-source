---
title: '[netfilter] Introduction to ebtables'
keywords: 'Linux,Netfilter,ebtables'
abbrlink: 35077
date: 2018-09-13 02:22:06
tags:
  - Network
  - Netfilter
  - Linux
  - Kernel
description: 本文是 iptables/ebtables 系列分享文的第一篇，會先著重於 iptables/ebtables 本身的架構，更準確的是 netfilter 的架構介紹，從 User-Space 到 Kernel-Space 的組成元件，並且簡單敘述一下整體的運作流程。最後開始介紹 ebtables 這個存在但是較少人知道的工具，不同於 iptables, ebtables 更專注於基於 MAC 地址的 Layer2 轉發。 文章最後介紹了 ebtables 的規則組成，並且將 ebtables 規則的處理順序以圖表的方式呈現，讓大家更容易理解在 Layer2 轉發時，該怎麼透過 `ebtables` 去設定相關的規則來處理封包。

---

## Preface

再之前的 `kubernetes server` 系列文中有稍微介紹過 `iptables` 的一些用法，以及如何透過 `iptables` 來完成 `kubernetes service`. 若有任何不熟悉的概念可以重新閱讀一次該系列文章.
- [[Kubernetes] How To Implement Kubernetes Service - ClusterIP](https://www.hwchiu.com/kubernetes-service-ii.html)


回到主題，這次想要跟大家慢慢介紹的就是 `iptables` 這個常見也常用的工具。
網路上其實已經可以搜尋到非常多關於 `iptables` 相關的文章。
不論是基本介紹，或是一些相關用法，其實都有滿多的資源可以學習，不過我認為這些文章都散落各地，所以想要整理一下這些資訊並且統整起來做一個一系列的`iptables`文章。
這個系列文的內容大致上如下
1. iptables/ebtables 的基本架構介紹，包含下列各種組成的概念
    - Target/Chain/Table/Match
2. 透過 `docker` 預設網路`Bridge`的情況來解釋，容器與外界網路，容器與容器彼此之間的網路傳輸，實際上再 `iptables` 中到底會怎麼運作，如果想要處理這些封包，該怎麼設定相關規則。
3. 介紹相關 `iptables` 常見的使用問題
4. 最後則是會跟大家介紹，如何自己手動撰寫一個 `iptables` 擴充模組，讓你的`iptables`擁有獨一無二的功能


本文則是系列文第一篇，將著重於`netfilter`的架構介紹，讓大家對於`netfilter` 的概念有個基本的認知與瞭解。
最後則會介紹一下 `ebtables` 的概念以及封包傳遞過程。

相關系列文章
- [[netfilter] Introduction to iptables](https://www.hwchiu.com/netfilter-eiptables-ii.html)
- [[netfilter] Dig Into Docker Bridge Network By iptables/ebtables](https://www.hwchiu.com/netfilter-eiptables-iii.html)


## Introduction
一般我們常常在講的 `iptables`，其實背後真正的專案以及相關技術都跟 `netfilter` 密切相關。

`netfilter` 的[官方網站](https://www.netfilter.org/) 上面有非常詳細的介紹，同時有非常多相關的專案。
譬如從使用者會使用的指令到給使用者開發的函式庫全部都該網站上面找到，而今天我們的主角 `iptables` 則是屬於`給使用者使用的指令`.

下圖是一個可以簡單描述整個 `iptables/netfilter` 的架構圖。
整個系統的組成分成三大部分，分別是 `user-space`,`kernel-space` 以及 `network interface cards`.

![Imgur](https://i.imgur.com/WlcgN7L.png)

### User Space
大家最常見也最熟悉的 `iptables` 以及比較少人知道的 `ebtalbes` 都是所謂的 `user-space` 的應用程式。
這類的應用程式提供了一個友善且易於操作的環境讓`使用者/系統管理者`有辦法去對`kernel`的 `netfilter` 進行操作。

除了 `netfilter` 專案本身提供的工具之外，使用者也可以自行的透過相關的程式語言函式庫來與 `kernel` 內的 `netfilter` 溝通。

與`kernel`的溝通方面，一般都是採用 `system call` 的方式來溝通，針對一些特別的應用甚至可以採用 `netlink` 的方式去接收封包來進行二次處理。

對二次處理有興趣的人可以搜尋 `NFQUEUE` 相關的資源


### Kernel Space
整個 `netfilter` 的精華與操作基本上都是在 `kernel` 層級去完成的。再此 `netfilter` 的子系統內，包含了所有使用者透過 `iptables` 所傳達的所有指令，譬如`丟棄所有來自a.b.c.d的封包` 這種類似的規則，全部都會存在 `kernel` 內。

這邊值得注意的是，因為 `kernel` 每次重新開機後，上次存在記憶體內的資訊都會消失，這意味者所有`iptables`所傳達的命令都會消失，因此才會有所謂的 `iptables save/restore` 等相關的指令然後搭配上 `systemd/upstart` 等機制來確保每次重新開機後相關的規則都會重新部屬上去。

### Network Interface Card
大家會使用 `iptables` 一定都是想要針對特定的網路封包進行一些處理，而這些網路封包必然伴隨網路設備 `Network Interface Card` 一同出現。
這些設備可以是實體的網路卡，也可以是系統上虛擬出來的，譬如`loopback`,`docker0(bridge0)`,`tuntap`,`veth` 等各式各樣的虛擬網卡。

當這些網卡跟 `kernel` 透過某種關係連結後，從這些網卡`進/出`的封包就會受到 `netfilter` 的影響，進而可以透過 `iptables` 所傳達的規則進行控制與約束


### Workflow
1. 網卡與系統掛勾，此時 `kernel` 知道有哪些網卡**Physical/Virtaul** 正在系統上
2. 使用者透過 `iptables/ebtables` 等工具將相關的規則寫入到 `kernel` 裡面，此時 `kernel netfilter subsystem` 內就會有對應的規則。
3. 當有任何封包從任何網卡內`進/出`時，這些封包就會被上述已經創立的規則進行比對，並且進行對應的動作。


## iptables/ebtables
有了上述的基本流程後，我們接下來要專注再`user-space` 的規則方面，瞭解該怎麼解讀每個規則。

大部分的人比較少聽到 `ebtalbes`, 然而為了更加理解容器本身的網路運作原理，因此這邊還是要跟大家介紹一下 `ebtalbes` 這個工具。

### Component
規則組成方面，基本上 `iptables/ebtables` 的概念是一致的，所以為了節省空間，決定一起介紹。

這邊我們先隨便拿一個常見的 `iptables` 規則來當做範例
```bash=
iptables -t nat -A POSTROUTING ! -s 10.244.0.0/16 -d 10.244.0.0/16 -j MASQUERADE
```

上述的規則用中文來說就是`我希望來源IP不屬於 10.244.0.0/16, 且目的IP屬於 10.244.0.0/16的封包，都進行 MASQUERADE(SNAT) 這個動作`

但是對於整個系統來說，該指令其實可以分成四大部分
1. Table: **-t nat**
2. Chain: **-A POSTROUTING**
3. Match: **! -s 10.244.0.0/16 -d 10.244.0.0/16**
4. TARGET: **-j MASQUERADE**

接下來會針對這四個部分進行說明與介紹

### Table
首先，我們已經知道整個 `iptables/ebtables/netfilter` 的運作是由各式各樣的規則所組成的。

然而每條規則的用途不會完全相同，為了妥善管理與使用，會希望將相同用途的規則放置在相同的 `Table` 裡面。

舉例來說，最常見也是大部分指令的預設值, `filter`
其功能就如同其名稱一樣，用來進行 `filter` 相關的規則，譬如將封包丟棄，讓封包通過等常見行為都放置在這個 `filter` 的 `table` 中。

針對上述的範例, `-t nat` 的意思就是這條規則要放置在 `nat` 這個`Table`之中。


### Chain
Chain 的概念比較複雜，分為所謂的 `Build-in Chain` 以及使用者自己創立的 `Chain`.

使用者創立的 `Chain` 比較偏向用來管理，而且必須要從 `Build-in  Chain` 這邊進去，所以這邊還是會比較專注於 `Build-in Chain` 的介紹。

Chain 相對於 `Table` 來說，我覺得可以想成是一個狀態點，該狀態是用來描述該封包當前的階段。

用比較口語話的說法就是這條`iptables`規則會再什麼時間點去封包進行匹配。舉 `INPUT chain` 來說，這個狀態就代表封包準備進入到系統後但是還沒有被對應的應用程式接收。

以底層的實現來說， `Table` 與 `Chain ` 彼此是互相連結綁定的。
只有特定相關邏輯的 `Table` 可以再特定時間點上 `Chain` 去組合成一個常見的 `iptables 規則`

譬如 `filter Table` 就可以運行再 `INPUT` Chain 但是卻不能運行再 `PREROUTING Chain`.

實際上會有哪些 `Chain` 以及哪些 `Table` 會因為 `iptables/ebtables` 本身的架構而有所不同，因此會在下面的部份再詳細介紹。

針對上述的範例 `-A POSTROUTING` 來看一下，他的意思就是我希望當前這條規則要放到 `POSTROUTING` 這條 `chain` 裡面，同時因為 `Chain` 裡面的規則本身是有**次序性的**, 所以再寫入規則的時候
可以使用 `-I (insert) 或是 -A (append)` 的概念來決定這條新規則的位置。

### Match
Match 的概念就非常簡單，每個規則都可以去描述符合該規則的封包格式，這部份除了預設的封包格式之外，也有不少擴充模組可以使用。
譬如最常見的就是
`-m tcp --dport 53` 這種針對 `TCP` 封包且目標連接埠是`53`的規則

回到上述的範例,**! -s 10.244.0.0/16 -d 10.244.0.0/16** 這個規則希望可以比對封包的來源/目的 `IP` 位置，同時藉由 `!` 這個符號來反轉比對結果。
因此就是所有來源不是 `10.244.0.0/16` 但是卻要送往 `10.244.0.0/16` 的封包。

此外，在 `Match` 方面，一條規則可以同時使用多個擴充模組來更加強化要比對的規則。

### Target
最後一個部分，就是當規則符合之後，要進行什麼樣的動作。
常見的有 `Acceptr`,`Drop` 這種給 `filter Table` 使用的動作。
複雜的甚至可以是修改封包格式，譬如 `Source Network Address Translation(SNAT), Destination Network Address Translation(DNAT)` 這種。

回到上述的範例, **-j MASQUERADE**，這邊會透過 `-j` 的方式描述後面要銜接的參數就是 `Target`, 而 `MASQUERADE` 則是一種讓`kernel`幫你維護的 `SNAT` 動作。


## ebtables
`ebtalbes` 相對於 `iptables` 來說使用的情境比較少，主要是其針對的目標範圍都著重於 `Layer2` 這層次，也就是 Ethernet Frame 這邊。

對於大部分的使用者以及管理者來說，通常都是以 `IP` 為基本單位來進行管理的，然而部分的網路元件設計者，譬如 [
Virtlet, a Kubernetes runtime server](https://github.com/Mirantis/virtlet/blob/master/docs/networking.md) 再其架構中就有使用到 `ebtables` 來幫忙完成特定的功能。

此外，為了下一篇文章能夠更理解整個 `docker bridge network` 的傳輸過程，對於 `ebtables` 還是要有點基本概念，這樣才可以對整體架構有個完整的認識

### Table
`Table` 方面，目前的 `ebtalbe` 總共有三張 `tables`, 分別是 `filter`,`nat` 以及 `broute`.

1. `filter` 則是專注於 `Accept/Drop` 相關過濾的行為
2. `nat`  這邊則專注於針對 `MacAddress` 相關的轉換
3. `broute` 這邊可以用來決定封包到底要進行 `Layer2 Bridging` 或是 `Layer3 Routing`。算是一個比較特別的規則。

### Chain
`Chain` 的話，再 `ebtables` 中總共有六條 `chain`.

- `brouting`: 這個 `chain` 的執行順序非常的早，基本上是`ebtalbes` 裡面最早的 `chain`. 只要當任何 `linux bridge` 上面的任何一個 `port` 有收到任何 `Frame` 進來後，就會到這個`Chain`裡面去進行比對. 這邊通常不太會去設定，而是都會依賴後續的 `Bridging Decision`透過`Mac Address` 去決定封包到底該怎麼走。

因為 `Linux Bridge` 本身會有 `STP(Spanning Tree Protocol)` 的運作，因此只有歸類於可轉發的連接埠才需要去進行這些封包比對。

- `input`: 如果今天封包的目標 `Mac Address` 是 `Linux Bridge` 本身的話，那封包就會進入到 `input chain` 來處理。最常見的範例就是 `docker` 容器想要對不同網段進行存取，會先進入到 `gateway` 也就是所謂的 `linux bridge` 本身。

- `output`:  針對要從 `Linux Bridge` 離開的封包都會進行處理的`Chain`，這類型的封包大致上有兩種可能。第一種是主機本身產生的封包，目的就是要從該`linux bridge`底下的某些 `Port` 轉發出去，或是該封包是從 `linux bridge` 的某些 `Port` 轉發到其他的 `Port`.
- `forward`: 針對會被`Linux Bridge`進行`Layer2 Bridging` 轉發的封包所進行的 `Chain`. 基本上同網段的容器間傳輸的封包都會到這個階段。
- `prerouting`: 這個 `Chian` 就是其名稱的解讀，`Pre-Routing`, 再封包進入到 `Linux Bridge` 後，但是還沒有碰到 `Bridging Decision` 前可以進行的階段。
- `postrouting`: 這個 `Chian` 就是其名稱的解讀，`Post-Routing`, 再封包準備離開 `Linux Bridge` 前，但是還沒有碰到 真正的透過網卡送出去前可以進行的階段。

其實對於 `ebtables` 來說，用 `pre-forwarding` 以及 `post-forwarding` 會更貼切，畢竟 `routing` 是更偏向 `Layer3` 路由方面的規則。

這邊因為 `iptables` 長期的習慣所以在命名方面也就遷就於此

### Match
在比對的規則來說， `ebtalbes` 專至於 `Layer2` 相關的處理，譬如 `unitcast/multicast/broadcast`，甚至是 `vlan/stp` 等
封包都可以處理。

此外，除了常見的封包內容外，也可以透過`-m`這個方式去使用擴充模組來達到更靈活的比對功能。


### Target
就如同前面所描述的，預設的 `Target` 其實都會跟對應的 `Table` 有關，譬如 `ACCEPT/DROP` 就會在 `filter/brouting` 這些`Table`.

雖然都叫做 `ACCEPT/DROP`, 其兩者的意思在 `filter/brouting` 的用途卻是不一樣的，有興趣的人可以直接參考 `man page` 來學習更多的用法與概念。

除此之外，也有許多的擴充模組提供更多強大的功能，譬如可以透過 `arp` 等相關的 `Target`, 針對 `ARP` 相關的封包直接讓系統幫你回應對應的 `ARP Reply`。


### Summary

看到這邊，我們用一張圖片，將上述的概念重新整合一次，將 `Table/Chain` 與 `ebtables` 進行整合。

![Imgur](https://i.imgur.com/q1SSG2l.png)

首先，關於該圖片中我大致上分成三個層次，分別
1. `user-space` 的任何網路應用程式
2. `kernel-space` 裡面偏向 `Layer3` 相關的處理，這部份其實是`iptables`的主戰場，但是因為整體架構的原因，這邊只會稍微帶過
3. `kernel-space` 裡面偏向 `Layer2` 相關的處理，這邊就是偏向 `ebtalbes` 的處理

首先當封包從跟`Linux Bridge`有關的網卡，左邊的`nic` 近來之後，首先會先進入到 `brouting` 這個 `chain`，`brouting` 這個 `chain` 裡面只支援 `brouintg Table`, 若有任何規則要將該封包直接透過`routing`處理的話，封包就會直接拉到 `kernel-space/layer3` 層級來處理， 這個範例中先用一個 `Magic` 的概念來代表任何跟 `Layer3` 有關的操作。


接者封包會跑到 `Prerouting` 這邊，該`chain`裡面只有`nat`可以執行，這意味到這個階段頂多只能對封包進行內容的修改，還沒有辦法丟棄。
這邊要注意，這時候還沒決定到底封包該怎麼轉送，所以可以透過修改封包的目標 `Mac Address` 來影響到底之後該怎麼轉送封包，因此這也是這邊 `Chian` 為什麼叫做 `PreRouting`.


接下來就會到所謂的 `Bridging Decision`. 這邊其實就會運行 `Learning Bridging` 相關的算法，根據 `Mac Address` 來決定轉發的方向，若目標對象是 `Linux Bridge` 本身，則會透過 `input chain` 一路往上送到 `Layer3` 去處理。
對於 `input` 來說，可以透過 `filter` 來進行封包的處理，決定哪些封包要過，哪些不要過。

如果只是該 `Linux Bridge` 下不同連接埠的轉發的話，就會走 `Forward` 這邊，最後透過 `Postrouting` 這邊進行後續的修改，最後就從目標的網卡將該封包送出去。

如果今天是主機上面的應用程式要送封包出去，且該封包目的地最後會透過 `Linux Bridge` 來轉發
則該封包最後經過 `Layer3` 神祕的處理後，最後會跑回到 `Layer2` 這邊，並且經過 `output chain` 以及 `postrouting` 依序的處理，最後也走到網卡出去。


本文到這邊我們先建立一個基本的概念，到底 `iptables/ebtables` 的組成元件以及對應的概念。
同時也稍微觀看了一下 `ebtables` 本身的封包傳輸流程。
然而只有單單的 `ebtables` 是沒有辦法理解以及解釋整個 `docker bridge network` 的運作及傳輸。
因此我們會在下篇文章以相同的觀點來分析 `iptables`, 並且將 `iptables` 以及 `ebtables` 給整合起來分析整體的運作流程。
到時候在分析 `docker bridge network` 的時候，能夠有更詳細的概念與背景去瞭解整體封包的傳輸過程




## Reference
- [ebtables man page](https://linux.die.net/man/8/ebtables)
- [ebtables/iptables interaction on a Linux-based bridge](http://ebtables.netfilter.org/br_fw_ia/br_fw_ia.html)

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