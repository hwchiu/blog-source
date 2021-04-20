---
title: Docker 網路入門篇(四) - 外界主動存取
keywords: 'Kubernetes,Network,Linux,Ubuntu'
tags:
  - Docker
  - Network
  - Kubernetes
description: 本篇文章探討 Docker Bridge 網路模型的運作過程，透過步驟去分析到底 docker -p 的背後運作原理
abbrlink: 46765
date: 2020-11-21 21:40:39
---

# 前言
本篇文章是 Docker 網路入門篇系列文最終篇，閱讀本文前要先有前面三篇文章的基本概念，因此還不夠熟悉的讀者可以再次閱讀前面三篇文章

[Docker Network - 網路模型](https://www.hwchiu.com/docker-network-model.html)
[Docker 網路入門篇(二) - Bridge 網路模型](https://www.hwchiu.com/docker-network-model-lab.html)
[Docker 網路入門篇(三) - 網路存取分析](https://www.hwchiu.com/docker-network-model-snat.html)
> 這系列的文章都會用比較使用者的角度來探討網路概念，比較不會去深度探討底層實作細節


# 本文
上篇文章中，我們主要分析的是如何讓我們創建的 Container 能夠存取外部網路，這過程中我們簡單的探討了關於 iptables 的用法，包含了
1. SNAT
2. Default Policy

透過 Source Network Address Translation (SNAT) 我們能夠讓外界的網路知道該如何把封包送回到我們的宿主機，接者宿主機內的 Kernel 會把封包給轉回到容器內。
而 Docker 安裝時會將預設的 iptables 的防火牆設定為白名單機制，沒有描述的封包一律丟棄，因此我們可以透過修改成黑名單機制，或是加上一條新的規則來允許容器封包可以連接外部。
然而，這種走法是 **容器主動存取外部網路，外部網路回覆請求**

而這篇文章則要來探討最後一個網路流向，如何讓 **外部網路，主動存取我們的容器**
最簡單的範例就是我們運行容器時，可以透過 **-p local_port:container_port** 的指令去告訴 docker 說，請幫我們設定一個網路轉發，當封包到達宿主機的  **local_port** 後，請幫忙將封包給轉送到容器內的 **container_port**。

透過這個機制，我們可以在系統上運行很多的 **nginx www server**，每個本身都專注於自己的 **80 port**，但是系統上透過不同的 **local_port** 來進行轉發。

因此接下來就來看看，這過程應該要怎麼實現，有哪些部分需要注意

# 環境
```
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.3 LTS
Release:        18.04
Codename:       bionic

$ uname -a
Linux k8s-dev 4.15.0-72-generic #81-Ubuntu SMP Tue Nov 26 12:20:02 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

$ docker --version
Docker version 19.03.13, build 4484c46d9d
```
實驗所有步驟都可以於 [GitHub Repo](https://github.com/technologynoteniu/bloglab/tree/main/docker_network_basic_4) 中找到

# 外界網路主動存取

為了讓本篇文章專注於如何讓 **外界能夠主動存取容器**，因此前幾篇文章設定的部分這邊就不會再提，可以直接透過運行這個[腳本](https://github.com/technologynoteniu/bloglab/blob/main/docker_network_basic_4/lab1.sh)來創建簡單環境

```bash=

docker run --privileged -d --network=none --name c1 hwchiu/netutils

sudo brctl addbr hwchiu0
sudo ifconfig hwchiu0 up
sudo ip link add dev c1-eth0 type veth

sudo ln -s /var/run/docker/netns /var/run/netns

c1_ns=$(docker inspect c1 | jq -r '.[0].NetworkSettings.SandboxID' | cut -c1-12)
sudo ip link set c1-eth0 netns ${c1_ns} name eth0
sudo docker exec c1 ifconfig -a

sudo brctl addif hwchiu0 veth0
sudo ifconfig veth0 up

sudo docker exec c1 ifconfig eth0 10.55.66.2 netmask 255.255.255.0 up

sudo ifconfig hwchiu0 10.55.66.1 netmask 255.255.255.0
sudo docker exec -it c1 ip route add default via 10.55.66.1
sudo iptables -t nat -I POSTROUTING -s 10.55.66.2/32 -o eth0 -j MASQUERADE
sudo iptables -t filter -I FORWARD -i hwchiu0 -o eth0 -j ACCEPT
sudo iptables -t filter -I FORWARD -i eth0 -o hwchiu0 -j ACCEPT
sudo iptables -t filter -I FORWARD -i hwchiu0 -o eth1 -j ACCEPT
sudo iptables -t filter -I FORWARD -i eth1 -o hwchiu0 -j ACCEPT
```

我們的目標就是打通這條路，最終會希望可以打造出跟 **docker -p local_port:contianer_port** 一樣的情形
![](https://i.imgur.com/TMqnQYg.jpg)


## 準備 Web Server
為了讓整個 Demo 順利與方便，我們會於準備好的容器中透過 **python** 創建一個 **web server**。


```bash=
$ docker exec -it c1 apt-get install -y python3

$ docker exec c1 python3 -m http.server&

$ curl 10.55.66.2:8000
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ascii">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".dockerenv">.dockerenv</a></li>
<li><a href="bin/">bin/</a></li>
<li><a href="boot/">boot/</a></li>
<li><a href="dev/">dev/</a></li>
<li><a href="entrypoint.bash">entrypoint.bash</a></li>
<li><a href="etc/">etc/</a></li>
<li><a href="home/">home/</a></li>
<li><a href="lib/">lib/</a></li>
<li><a href="lib64/">lib64/</a></li>
<li><a href="media/">media/</a></li>
<li><a href="mnt/">mnt/</a></li>
<li><a href="opt/">opt/</a></li>
<li><a href="proc/">proc/</a></li>
<li><a href="root/">root/</a></li>
<li><a href="run/">run/</a></li>
<li><a href="sbin/">sbin/</a></li>
<li><a href="srv/">srv/</a></li>
<li><a href="sys/">sys/</a></li>
<li><a href="tmp/">tmp/</a></li>
<li><a href="usr/">usr/</a></li>
<li><a href="var/">var/</a></li>
</ul>
<hr>
</body>
</html>
```

透過上述的指令我們已經創建了一個網頁伺服器，並且透過 **curl** 順利存取，確保該服務沒問題。

因此接下來，我們要做的就是如何透過 iptables 讓外界網路能夠存取該容器

## DNAT

基本上網路概念全部都一樣，我們要探討的問題都是一樣的，就是**封包該怎麼送**，**誰來負責轉發**。

今天外界網路要存取我們內部的容器，想法就是 **就是封包要先到達宿主機，接下來宿主機想辦法幫你轉送到內部容器**。相對於 **SNAT**，這邊要採用的策略是 Destination Network Address Translation(DNAT)，將封包的目標地址進行修改。

接下來我們來進行一個範例，我們希望做到 **-p 2345:8000** 的範例，將送到本地的 **2345** 送到容器內的 **8000 port** 內。

```bash=
$ sudo iptables -t nat -A PREROUTING -p tcp -m tcp --dport 2345 -j DNAT --to-destination 10.55.66.2:8000
```

上述的指令概念意思是，看到 TCP 的封包，且目標是 2345 port，則透過 DNAT 幫我把封包的目標 IP 改成 10.55.66.2, Port 改成 8000

接下來到其他環境，對宿主機使用 curl 來存取看看 port 2345
```bash=
$ curl 172.17.9.103:2345
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ascii">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".dockerenv">.dockerenv</a></li>
<li><a href="bin/">bin/</a></li>
<li><a href="boot/">boot/</a></li>
<li><a href="dev/">dev/</a></li>
<li><a href="entrypoint.bash">entrypoint.bash</a></li>
<li><a href="etc/">etc/</a></li>
<li><a href="home/">home/</a></li>
<li><a href="lib/">lib/</a></li>
<li><a href="lib64/">lib64/</a></li>
<li><a href="media/">media/</a></li>
<li><a href="mnt/">mnt/</a></li>
<li><a href="opt/">opt/</a></li>
<li><a href="proc/">proc/</a></li>
<li><a href="root/">root/</a></li>
<li><a href="run/">run/</a></li>
<li><a href="sbin/">sbin/</a></li>
<li><a href="srv/">srv/</a></li>
<li><a href="sys/">sys/</a></li>
<li><a href="tmp/">tmp/</a></li>
<li><a href="usr/">usr/</a></li>
<li><a href="var/">var/</a></li>
</ul>
<hr>
</body>
</html>
```

到這邊為止，基本上我們幾乎實現 **docker -p** 的功能，能夠讓外界存取到內部容器了，其流程如下
![](https://i.imgur.com/mmYV2cK.jpg)


## 其他問題
看似完美解決問題，其實背後還是有一些東西要處理

### 本地存取
完成上述規則後，我們嘗試於宿主機內針對設定的 **2345** 去進行存取

```bash=
$ curl 172.17.9.103:2345
curl: (7) Failed to connect to 172.17.9.103 port 2345: Connection refused

$ curl 127.0.0.1:2345
curl: (7) Failed to connect to 127.0.0.1 port 2345: Connection refused

$ curl localhost:2345
curl: (7) Failed to connect to localhost port 2345: Connection refused
```

結果看起來我們設定的 iptables 規則對於這種本地直接存取自己是有問題的，但是如果使用的是 **docker -p** 去創造服務的話，會發現其實這些是通的，那到底 **docker** 是怎麼解決這些問題?

### 重複使用
第二個要解決的問題是，如果今天有程式想要於系統中去使用 **port 2345**，這時候問題就來了。

我們事先透過 iptables 去描述 DNAT，將 2345 轉為容器內80，這部分其實對於 **socket programming** 來說是不知情的，所以任何人都還是有機會去使用本地機器的 **2345 port**。

在這種情況下，會跑出一個疑問，請問送到 **2345** 的封包到底是要依據我們的規則去轉發給容器，還是要送給這個使用 **2345** 的本地程式?


### 解決方式
為了解決這兩個問題，這邊我們先看個範例
```bash=
$ docker run -d -p 12345:80 nginx

$ curl localhost:12345
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

$ ps auxw | grep docker-proxy
root     20060  0.0  0.1 552996  4060 ?        Sl   05:20   0:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 12345 -container-ip 172.18.0.2 -container-port 80

$ sudo netstat -anlpt | grep 12345
tcp6       0      0 :::12345                :::*                    LISTEN      20060/docker-proxy
....
```

這邊我們透過 **docker** 快速起一個 **nginx** 的服務，並且設定 **12345:80** 的關係。

可以看到如果 **docker** 的話，我們是可以透過 **localhost:12345** 來存取容器的，這是我們剛剛自己設定 iptables 辦不到的。

解法是，**docker** 於本地端偷偷部署了一個新的程式，叫做 **docker-proxy**，可以看一下其運行的參數

```bash=
-proto tcp -host-ip 0.0.0.0 -host-port 12345 -container-ip 172.18.0.2 -container-port 80
```

有沒有覺得這些內容跟我們 **iptables** 的規則很類似?
```bash=
iptables -t nat -A PREROUTING -p tcp -m tcp --dport 2345 -j DNAT --to-destination 10.55.66.2:8000
```

1. -proto tcp  <----> -p tcp
2. -host-port <----> -m tcp --dport
3. --container-ip, --contianer-port <-----> -j DNAT --to-destination

所以其實這是一個，針對 **loclahost** 流量而設計的 Proxy 程式，幫忙處理 DNAT，將封包轉送到內部去。

此外，這個應用程式本身也會去聽 **12345** 這個 port，因此未來如果有任何應用程式想要註冊 **12345** 的話，就會沒有辦法註冊，因為已經被搶走


# 總結

1. **docker -p** 做法非常簡單，就是整合了 iptables DNAT 以及 docker-proxy
2. 透過 iptables DNAT 的功能，我們可以讓外部網路透過宿主機來存取內部容器，然而這種情況下，你是沒有辦法於宿主機內透過一樣的方式去存取容器的，這部分的原因是 **iptables** 的執行點跟 **localhost** 這種直接存取是有落差的
3. docker透過 docker-proxy 的方式來彌補上述的缺陷，專門用來處理本地封包
4. 同時， docker-proxy 也會佔據本地的 port，避免未來有其他的程式想要使用相同的 port 而造成混淆與服務不如預期。


到這邊為止，我們透過四篇文章來探討了 Docker 的網路模型，並且嘗試透過手工的方式，打造一個相同的 Bridge 網路模型。過程中，我們也一併探討要如何讓容器擁有網路存取能力，不論是容器間互相存取，容器主動對外存取或是外界主動存取容器。
這些步驟背後原理非常複雜，實際的實作細節牽扯到 Linux Kernel 內的很多元件，包含了 netfilter, conntrack, iptables 等，而本系列文目標是從概念出發，從大方向去理解到底怎麼運作，**docker** 怎麼做，有了這些基本概念後
未來有興趣去學習底層運作時，也比較會有方向知道該怎麼下手。


# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
