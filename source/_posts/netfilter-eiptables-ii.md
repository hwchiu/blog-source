---
title: '[netfilter] Introduction to iptables'
keywords: 'Linux,Netfilter,iptables,chain'
abbrlink: 11657
date: 2018-09-15 14:18:36
tags:
  - Network
  - Netfilter
  - Linux
  - Kernel
description:
---

這次想要跟大家慢慢介紹的就是 `iptables` 這個常見也常用的工具。
網路上其實已經可以搜尋到非常多關於 `iptables` 相關的文章。
不論是基本介紹，或是一些相關用法，其實都有滿多的資源可以學習，不過我認為這些文章都散落各地，所以想要整理一下這些資訊並且統整起來做一個一系列的`iptables` 文章。

這個系列文的內容大致上如下
1. iptables/ebtables 的基本架構介紹，包含下列各種組成的概念
    - Target/Chain/Table/Match
2. 透過 `docker` 預設網路`Bridge`的情況來解釋，容器與外界網路，容器與容器彼此之間的網路傳輸，實際上再 `iptables` 中到底會怎麼運作，如果想要處理這些封包，該怎麼設定相關規則。
3. 介紹相關 `iptables` 常見的使用問題
4. 最後則是會跟大家介紹，如何自己手動撰寫一個 `iptables` 擴充模組，讓你的`iptables`擁有獨一無二的功能


本文延續前一篇 `ebtables` 的介紹，將使用相同的概念來闡述 `iptables(ipv4)` 的概念，包含了 `Tarble/Chain/Match/Target` 等功能。

相關系列文章
- [[netfilter] Introduction to ebtables](https://www.hwchiu.com/netfilter-eiptables-i.html)

<!--more-->

## Introduction
為了能夠更充分理解本文所描式的各個觀念，強烈建議先閱讀[前篇文章](https://www.hwchiu.com/netfilter-eiptables-i.html) 來理解整個規則裡面的四大部分，`Table/Chain/Match/Target`

這邊再次做個快速的複習
- Table: 相同用途的 `rules` 會放在相同的 `Table` 中，常見的有用來當防火牆的 `filter`，或是修改封包內容的 `nat`.
- Chain: 封包比對的時間點，不同時間點下能夠進行不同的動作。這意味每個`Chain` 下能夠搭配的 `Table` 是有限制的。 
- Match: 每個規則都要描述當前規則希望匹配的封包內容，除了基本的比對欄位外，還可以用各式各樣擴充模組來匹配不同的封包內容。
- Target: 當封包比對成功後，要執行什麼樣的動作，不同於 `Match` 可比對多個內容，每個規則都只能採用一個 `Target` 來採取動作。


## iptables
`iptables` 的用途非常的廣，以 `docker` 為範例來說，從基本的容器對外上網，這邊會需要 `SNAT` 來轉換封包。或是透過 `docker run -p 1234:80 xxxx` 這種方式讓外界能夠透過 `DNAT` 的方式來存取容器內的特定連接埠。

上述的這些顯而易見的操作實際上背後是牽扯到了非常複雜的封包傳輸，為了理解這部份，我們要先來檢視一下 `iptables` 裡面四大部分的介紹

### Table
`Table` 方面，目前的 `iptables` 總共有五張 `tables`, 分別是 `filter`,`nat`,`raw`,`mangle` 以及 `security`.

1. `filter`: 跟 `ebtables` 一樣， `filter Table` 也是 `iptables` 系列指令的預設 table, 用來存放如 `ACCEPT/DROP` 等相關防火牆功能的規則。
2. `nat`: 就是如同其名稱一樣，`Network Address Translation(nat)`，對於來源或是目的的 `IP` 地址進行修改的動作都是再這邊進行的。
{% note info %}
實際上再 `Linux Kernel` 內有一套叫做 `conntrack` 的機制去維護所有經過本機的網路連線。
基本上只有新建立的連線才會進入到 `nat` 這個 `table` 去處理。
畢竟以 `SNAT` 這種會需要動態產生一個 `Port` 來進行轉發的動作，其實每條連線只要進行一次就好，後續該連線的封包就讓 `kernel` 幫你默默的執行就好。

之後有機會再來討論一下 `conntrack` 的機制與架構，以及其能夠提供什麼樣的資訊給系統管理者/使用者
{% endnote %}
3. `raw`: 這個 `chain` 比較少使用，其用途是用來特別處理不想要讓 `kernel`: 幫你管理 `conntrack` 的封包。
4. `mangle`: 除了`nat`能夠修改封包的 `IP` 地址外， `mangle` 也會用來進行一些封包的修改。然而其修改會比較偏向一些 `metadata` 標籤概念的欄位，讓其他的規則可以透過檢視這些標籤來得知該封包先前有符合某些條件，藉由這些更多的條件判斷來決定該怎麼處理封包。
{% note info %}
舉例來說，再 `iptables` 裡面有所謂的 `mark` 的概念，這個`32bit`的欄位並不屬於 OSI 裡面的任何一層的封包格式，而是 `linux kernel` 裡面用 `sk_buff` 該描述封包結構中自己新增的欄位。

透過這個欄位我們可以在不同的階段去追蹤相同的封包，來達到更複雜的處理。

譬如再 `FORWRAD Chain` 我想要知道這個封包是不是之前有再 `PREROUUTING`  被處理過，就可以用該 `mark` 來處理。
{% endnote %}
6. `security`: 這個 `table` 更少出現，必須伴隨者 `SELinux` 的使用來提供更多安全相關的功能，主要牽扯到 **Mandatory Access Control (MAC)** 規則以及 **Discretionary Access Control (DAC)** 這兩者的管理，有興趣的可以看看最初的 [commit](https://lwn.net/Articles/267140/)。

上述裡面，基本上 `raw/mangle/security` 這三個 `table` 比較少使用，所以後續會比較著重在 `filter/nat` 這兩個 `table` 為主。 
 
### Chain
`Chain` 的話，再 `ibtables` 中總共有五條 `chain`.


- `PREROUTING`: 這個 `Chian` 就是其名稱的解讀，`Pre-Routing`, 再封包進入到 `Linux Kernel` 後，但是還沒有碰到 `Routing Decision` 前可以進行的階段。

{% note info %}
這邊舉一個現實會使用到 `PREROUTING` 的使用情境，很多人在家裡可能會有架設 `server/nas` 等各種服務的可能，然而因為 `IP` 地址數量的限制，這些背後的機器都會使用私有的 `IP` 地址，譬如 `192.168.0.0/16`, 這種情況下為了讓外界能夠順利的存取到這些內部的 `server/nas`，常見的作法都是會在家裡對外上網的那台 `router` 設定譬如 `PortFORWARDing/虛擬伺服器` 等功能， 將特定的連接埠轉發到內部 `server` 的私有`IP`地址及連接埠。

這功能實際上就是在 `PREROUTING` 這個階段會進行 `DNAT`，將封包的目的`IP`位址與連接埠都轉換到內部`server`的`IP`地址與連接埠。
最後透過 `Routing Decision` 來往後轉發

{% endnote %}


- `INPUT`: 如果該封包根據 `Routing Decision` 後封包是要進入到本機系統，譬如系統上的應用程式，譬如 `www server`。 則`INPUT`就是查詢完畢到封包被應用程式接收的中間階段。

{% note info %}
如果今天機器上架設了一個 `nginx server`, 並且聽再 `0.0.0.0:80`. 則任何送到該機器網卡上面且連接埠是`80` 的封包最後都會經過 `INPUT chain` 來處理。 所以也可以在這邊透過其他的選項丟棄掉不想要連接到 `nginx server` 的封包。 
{% endnote %}


- `FORWARD`: 如果該封包根據 `Routing Decision` 後封包是要幫忙轉發。則`FORWARD`就是查詢完畢到封包要從網卡送出去的中間階段。

{% note info %}
實際上，預設的 `linux kernel` 是沒有 `FORWARD` 的功能的，必須要將 `kernel` 關於 `ip_FORWARD` 的開關打開才可以使用。
所以才會看到很多篇文章都在講解需要 `echo 1 > /proc/sys/net/ipv4/ip_FORWARD` 這種方式打開 `kernel` 內關於轉發的功能。

詳細的程式碼有興趣可以參考下列連結,(我這邊隨便找了一個 `Linux Kernel 4.3` 的程式碼)
1. [Read The ip_FORWARD](https://elixir.bootlin.com/linux/v4.3/source/net/ipv4/devinet.c#L2267)
2. [Check the device ip_FORWARD config](https://elixir.bootlin.com/linux/v4.3/source/include/linux/inetdevice.h#L92)
3. [Check the config to decide the routing](
https://elixir.bootlin.com/linux/v4.3/source/net/ipv4/route.c#L1761)

{% endnote %}

- `OUTPUT`:  針對要從 `Linux Kernel` 離開的封包都會進行處理的階段，這類型的封包是主機本身產生的封包，目的就是要從某些網卡轉發出去。 舉例來說系統上的 `nginx www server` 要回應使用者的需求，這些回應的封包就會走 `OUTPUT chain` 出去。


- `POSTROUTING`: 這個 `Chian` 就是其名稱的解讀，`Post-Routing`, 再封包準備從系統出去前，但是還沒有碰到真正的透過網卡送出去前可以進行的階段。

{% note info %}
這邊繼續使用家裡架設的 `server/nas` 當作範例，因為`IP`地址不夠的問題，所以內部這些`server/nas`要出去的封包其`來源IP`地址必須要修改成對外`Router`的`IP`地址。
而這個行為我們稱為所謂的 `Source Network Address Translation (SNAT)`，而這個操作都是在 `POSTROUTING` 這邊去執行的。
{% endnote %}

### Match
在比對的規則來說， `iptables` 專至於 `Layer3` 相關的處理，譬如 `IP` 的來源/目的地址，以及當前封包使用的`Layer4`協定，譬如
tcp, udp, udplite, icmp, esp, ah, sctp。

此外，除了常見的封包內容外，也可以透過`-m`這個方式去使用擴充模組來達到更靈活的比對功能。
譬如常見的 `-m tcp --dport=1234` 這個額外的 `tcp` 模組能夠檢查 `TCP` 封包裡面的欄位。
因為原生的 iptables 只有檢查到所謂的 `protocol` 協議而已，並沒有再細部的去解析封包內容，因此若需要細緻到該協議的內容，都需要依賴擴充模組來完成。


### Target
就如同前面所描述的，預設的 `Target` 其實都會跟對應的 `Table` 有關，譬如 `ACCEPT/DROP` 就會在 `filter` 這些`Table`. 

再 `iptagles` 裡面有四個預設的 `Target`
1. ACCEPT: 該封包判定通過，直接離開這個 Chain.
2. DROP: 丟棄該封包，直接離開這個 Chain.
3. QUEUE: 可以把封包從 `kernel-space` 透過 `netlink` 的方式送到 `user-space` 去後搭配 `DPI(Deep packet inspection)` 進行封包檢測來判斷當前封包的類型與種類
5. OTHER_CHAIN: 使用者可以自己創見新的 `chain` 然後透過 `-J $CHAIN_NAME` 的方式讓封包跳到不同的 `custom_chain` 去進行比對。
4. RETURN: 直接返回上一層的 `chain`, 通常是會搭配 `-j $CHAIN_NAME` 一起使用。

此外再 `iptables` 有非常多有趣的 `Target` 可以執行
1. SNAT/DNAT: 這種修改封包 `IP` 地址的行為
2. NFQUEUE: 擴充原先的 `QUEUE`，提供更多的 `queue number` 供 `user-space` 選擇。
3. log: 單純記錄封包資訊，並且從 `kernel` 輸出，可以傭 `dmesg` 去觀察該記錄。由於該 `Target` 的實作，其本身並不會做到類似 `ACCEPT/DROP` 這種馬上決定該封包去留的行為，而是會繼續讓封包往下一個規則嘗試比對。

{% note info %}
下一篇文章就會大量使用到 `log` 這個 `target` 來幫助我們觀察再容器間封包傳輸時，到底有哪些 `iptables/ebtables` 會被呼叫到。
{% endnote %}


### Summary

有了上述的基本概念後，我們把這些概念重新整合一次，將 `Table/Chain` 與 `iptables` 進行整合，同時為了簡單清楚，我們就只專注於 `nat/filter` 這兩張 `Table` 即可。


![Imgur](https://i.imgur.com/BFqDVwY.png)

首先，當封包從網卡進入後，首先會經過 `conntrack` 的管理，讓系統幫你進行連線追蹤的相關工作。

{% note info %}
這邊的說法都是精簡的，因為去掉了 `raw/mangle/security` 這些 `Table` 的關係，實際上 `raw` 本身的運作會比 `conntrack` 還要快。
{% endnote %}

接者就是所謂的 `PREROUTING`, 再系統根據封包的目的地`IP`地址進行選擇前，我們可以在 `PREROUTING` 透過 `DNAT` 的方式修改封包的目的`IP`地址，藉此改變封包的傳送對象。

最後就是所謂的 `Routing Decision` 了，這部份會在 `kernel` 內透過查詢 `routing table` 的方式

{% note info %}
可以透過 `ip route`, `route` 等相關的指令查詢系統上當前的 `routing` 規則。
{% endnote %}

`Routing` 查詢完畢之後，會有兩個走向，一個是將封包透過 `Socket` 的方式讓 `上層的應用程式` 去收取封包，這種情況下就會走過 `INPUT chain` 的階段處理。 管理者就可以在 `INPUT` 這邊實現簡單的防火牆，來針對特定的封包給予通過或是丟棄。

若查詢 `routing` 後決定要將封包轉發同時系統也有透過 `/proc/xxxxxx/ip_FORWARD` 進行設定，則此時就該封包就會開始進入到 `FORWARD` 這個階段進行處理, 此時也可以透過 `filter` 進行簡單的防火牆過濾，決定封包的去留。

走完 `FORWARD` 後就是所謂的 `POSTROUTING` 了，這邊可以進行所謂的 `SANT`, 將封包的來源 `IP` 地址修改以順利讓該封包能夠建立一條順利的網路連線。

{% note info %}
實際上，再 `iptables` 的規則中，有兩種的 `SNAT` 的實現方法，分別是 `-j SNAT xxx.xxx.xxx.xxx:xxx 以及 -j MASQUERDAE`.

因為 `SNAT` 再運作的時候其實需要考慮`連接埠`的轉換，每一條出去的連線都要搭配一個`連接埠`來作為回傳連線的匹配對象，所以傳統的 `SNAT Targer` 需要特別指定該次 `SNAT` 轉換後用的`IP`地址與連接埠。
不過這種情況實在是會讓整個系統變得不好用，所以後來發展出了 `MASQUERADE` 這種動態 `SNAT` 的方式讓 `kernel` 自己幫你選擇要使用的 `IP` 以及連接埠。
{% endnote %}

最後，若使用者的網路應用程式需要往外傳送封包，則該封包會先進入到 `OUTPUT Chain`, 這邊也可以透過 `filter` 進行防火牆的操作。
最後封包就會走入 `POSTROUTING` 進行後續的處理。

## Summary With Ebtables.
前述我們已經看到了 `iptables` 的運作流程，而前篇文章我們也看過了 `ebtables` 的運作流程。
現在我們需要將這兩者的邏輯給結合，構造出一個更複雜的網路系統。


![Imgur](https://i.imgur.com/q7hUk0X.png)

這張圖裡面我們依然分成三個部分來看待，分別是 `User-Application`, `Kernel-Space/Layer3` 以及 `Kernel-Space/Layer2`.

然後圖中針對 `iptables` 以及 `ebtalbes` 使用不同的顏色來表示彼此的 `Chain/Table`。

接下來要來好好的解釋這張圖的概念，在開始前我們先有一些相關的理解。
1. `iptables` 無所不在，縱使在 `Layer2 Bridging` 的世界中，還是會牽扯到 `iptables` 的運作。(**實際原因我不清楚，但是我想跟 `conntrack` 有關，任何的封包連線 `linux kernel` 都想要追蹤**, 此外，我想與透過 `IP` 地址方式去操作管理平易近人也有關)
2. 所有封包的 **起頭/終點** 一定是 **(網卡/應用程式)**,中間會經過 `Layer2` 也會經過 `Layer3`，這部份完全看你**封包來源網卡**與**封包目標網卡**屬於什麼層級而決定怎麼走

好了，我們可以好好的來重新審視這張圖，一開始我們就先從封包的進入點，也就是網卡這邊來看。
首先當封包進入網卡的時候，會先進入所謂的 `Bridge Check` 這個階段，這時候會決定封包要走到 `Layer3` 處理，還是 `Layer2` 處理。底下會針對這兩個 `Case` 探討

{% note info %}
其實這個階段非常有趣，各位可以想想看，當你看到一個封包，你怎麼知道這個封包到底是要 `Routing` 還是要 `Bridge`?
實際上在 `Linux Kernel` 來說，是透過所謂的 `netdev_rx_handler_register` 來註冊每張網卡收到封包後該怎麼處理。 以 `Linux Bridge` 來看，當透過 `brctl addif br1 xxx` 這個方式把 `xxx` 加入到 `br1` 這個 `bridge` 時，就會把 `xxx` 這個網卡的接收封包函式設定成 `bridge` 有關，所以之後近來的封包就會走 `Layer2` 的方式去跑，反之亦然其他按照相同道理就會走 `Layer3` 的流程。

有興趣觀看原始碼的可以參考下列連結
[register handler function](https://elixir.bootlin.com/linux/latest/source/net/bridge/br_if.c#L560)
[call ebtables](https://elixir.bootlin.com/linux/latest/source/net/bridge/br_input.c#L282)
{% endnote %}

### Layer3
如果今天封包走到了 `Layer3` 這邊來處理，那處理的流程基本上就跟本文前半部分描述的雷同，唯一不同點只有當進行完畢 `Routing Decision` 後，在選擇 `FORWARDd` 的階段，若轉送目的地網卡對應到的是屬於本機上面的 `Linux Bridge` 網卡，則封包最後又會走到 `Layer2` 那層，在這情況下就會在經過 `iptables` 後又會馬上轉接 `ebtables`，最後就會送到網卡出去。
        
### Layer2

如果封包一開始進入點就是 `Linux Bridge` 的網卡，這時候可以在 `brouting chain` 進行一次檢查，如果這時候有透過 `ebtables`特別將封包送到 `Layer3` 處理的話，那流程就會如同上述一樣，從 `conntrack` 一路走到 `Routing Decision`.
如果繼續決定走 `layer2` 來處理封包的話，那流程就會跟前篇文章講解 `ebtables` 敘述的流程一樣。只是這邊要特別注意的是，實際上 `iptables` 也會混雜在 `layer2` 的處理過程中，所以在真正進行 `Bridge Decision` 前也會遇到 `iptables PREROUTING` 進行處理。

如果透過 `Bridge Decision` 查詢目標的 `MAC Address` 後決定將封包轉送到 `Linux Bridge` 本身的話，那最後就會走向 `Layer3` 上層的走法，否則則會繼續在 `Layer2` 這邊將封包往其他的 `Bridge Port` 去轉發。

在 `Bridge Port` 轉發的過程中，也是會牽扯到 `iptables` 相關的規則。所以若只是單純的兩個 `Bridge` 底下的封包互相轉傳的話，其實也是可以透過 `iptables` 使用 `IP` 去控制封包轉送，或是透過 `ebtables` 透過 `MAC Adddress` 去控制封包轉送。


### Application

最後來看，如果是從本機應用程式送出去封包的走向，首先封包一定會經過 `Layer3` 相關的轉發，經過 `Routing Decision` 時就會知道目的地的網卡為何，如果目標網卡是屬於 `Linux Bridge`, 則最後該封包又會一路走到 `Layer2` 的部份，此時又可以透過 `iptables/ebtalbes` 兩者來處理封包。

如果目的網卡不是上述的，那基本上就會直接走完 `iptables` 的過程，最後透過網卡轉發出去。

{% note info %}
其實比較正確的比對方式應該是該網卡本身會怎麼處理封包，在 `Linux Kernel` 裡面會針對每個網卡**net device**去設定相關的收送函式，當有封包要從該網卡送出去時就會呼叫對應的函式，這時候裡面就會決定應該要怎麼處理封包，進而去呼叫對應的 `iptables/ebtalbes` 相關的處理。
所以一些特別的網卡，不論是 `IPSec/VXlan/Tun/Tap` 等實際上怎麼運行都還是要看 `kernel` 內真正的實作來決定到底封包會怎麼走。
{% endnote %}

到這邊已經將 `iptables` 以及 `ebtables` 兩者的關係給結合起來，可以觀察到實際上會經過的規則是非常的多。
下篇文章我們會嘗試使用真正的容器環境，搭配一些擴充模組來實際觀察這些容器不同方向的封包傳輸實際上會牽扯到哪些相關的 `TABLE/CHAIN`.


## Reference
- [ebtables man page](https://linux.die.net/man/8/ebtables)
- [ebtables/iptables interaction on a Linux-based bridge](http://ebtables.netfilter.org/br_fw_ia/br_fw_ia.html)
