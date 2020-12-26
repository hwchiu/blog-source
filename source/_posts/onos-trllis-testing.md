---
title: ONOS Trellis Testing
keywords: 'SDN,Network,Linux,Ubuntu,ONF,ONOS,Trellis,Mininet'
tags:
  - SDN
  - Network
  - ONF
  - ONOS
  - Trellis
  - Ubuntu
  - Mininet
abbrlink: 57907
date: 2017-12-11 14:14:56
description:
---
CORD-Trellis Example
====================
Trellis 是 ONF 於 2017 年推出的 Network Architecture Solution，整個架構就於 ONOS SDN Controller 以及 Openflow-Enable switch.

本篇文章主要是在於如何透過 ONF 提供個環境快速搭建一個 Trellis 的測試環境。
<!--more-->

Introduction
============
關於 **Trellis** 相關 script 的專案真正的位置是放在 ONF 內部的 [Project](https://gerrit.onosproject.org/#/q/project:routing)，不過你也可以在 [Github](https://github.com/opennetworkinglab/routing/blob/master/trellis/README.md)看到 Mirror 的版本。
在該 **README** 有提到該如何架設一個測試環境，因此本篇文章的內容就會基於該 **README** 去架設一個測試環境。

Environment
===========
基本上該測試環境是基於 **mininet** 與 **ONOS** 來部署的，所以在機器的數量上面，最少需要一台機器，最多沒有限制,因為 **ONOS** 本身可以是可以同時跑起多台的 **SDN Controller**，可以架設一個 **Cluster** 的環境然後與 **Mininet** 互連。
不過為了方便測試，並沒有需要架設到這麼多的 **ONOS SDN Controller**。
因此在本環境中，決定採用兩台機器即可。
所需環境
1. Two Ubuntu Machine
    - One for ONOS Controller (VM1)
    - One for Mininet emulator (VM2)
2. 於 VM1 上面請先安裝好 ONOS Controller, 詳細的安裝步驟可以參考 ONF 本身的 [wiki](https://wiki.onosproject.org/display/ONOS/Administrator+Guide), 這邊可以分成使用者跟開發者兩種運行方法，若是開發者本身會要你抓取 ONOS 的 source code，並且透過 **buck** 來進行建置與運行。此外，也可以直接使用 **ONOS** [docker image](https://wiki.onosproject.org/display/ONOS/Running+the+published+Docker+ONOS+images) 來運行 **ONOS**。
3. 於 VM2 上面安裝 Ubuntu 16.04 的環境，接下來就可以參考 **README** 的步驟來設置。


Steps
=====
#### VM1 (ONOS)
- 為了更放便控制於 ONOS 上面運行的 application，可以透過環境變數 `ONOS_APPS` 直接設定要運行的 apps。
- 透過下列指令控制要運行的 app.
``` sh
export ONOS_APPS=drivers,openflow,segmentrouting,fpm,dhcprelay,netcfghostprovider,routeradvertisement
```
- 運行起 ONOS Controller
- 另外開視窗，運行 ONOS Cli 工具，透過 `apps -a -s` 檢查運行的 apps 是否與上述吻合
- 下載 [trellis](https://github.com/opennetworkinglab/routing/blob/master/trellis/trellis.json) 相關設定檔案，並且透過下列工具將該設定檔寫入到 ONOS 中，其中 **<onos-ip>** 則是本機端的 IP address (此 IP 要讓 VM2 能夠存取得到)
``` sh
onos-netcfg <onos-ip> routing/trellis/trellis.json
```

#### VM2 (Mininet)
- 安裝相關軟體
```sh
sudo apt-get update
sudo apt-get install -y gawk texinfo python-pip build-essential iptables automake autoconf libtool
sudo pip install -U pip
sudo pip install ipaddress
sudo apt-get install isc-dhcp-server
sudo apt-get install mininet
```
- 安裝完畢後，魷魚 **Trellis** 架構內支援使用 **Quagga** 來當外部 BPG 溝通的橋樑，因此我們需要在本機上安裝 **Quagga**。
- 這邊要特別注意，在預設情況下, **quagga** 本身會期望運行的使用者名稱為 **quagga**，同時你也要幫`--sysconfdir`以及`--localstatedir` 這兩個位置的資料夾設定全縣，讓 **quagga** 此使用者有權限可以寫入。
- 若是單純測試的話，可以在 **configure** 的時候加入兩個選項 **--enable-user=root --enable-group=root**, 這樣 **Quagga** 相關應用程式就會採用 **root** 的身份去運行了。
``` sh
git clone -b onos-1.11 https://gerrit.opencord.org/quagga
cd quagga
./bootstrap.sh
./configure --enable-fpm --sbindir=/usr/lib/quagga
make
sudo make install
cd ..
```
- 接下來要修改本地端的檔案，讓我們的 mininet/Zebra 相關的應用程式能夠跟 ONOS 連接得到，所以請修改下列兩個檔案
    - routing/trellis/trellis.py
    - routing/trellis/zebradbgp1.conf
- 於 **routing/trellis/trellis.py**，請將下列三行指令中的後面兩行，並且將第一行指令中的 **IP** 換成 VM1 的 **IP**
```python
net.addController(RemoteController('c0', ip='<onos-ip>'))
#net.addController(RemoteController('c1', ip='192.168.56.12'))
#net.addController(RemoteController('c2', ip='192.168.56.13'))
```
- 於 **routing/trellis/zebradbgp1.conf** 裡面將 **IP** 的部分也換成 VM1 的 **IP**
``` sh
fpm connection ip <onos-ip> port 2620
```
- 接下來要將系統上 kernel 的保護機制 **app armor** 給關閉，執行下列指令
```bash
ln -s /etc/apparmor.d/usr.sbin.dhcpd /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.dhcpd
```
- 最後要運行時，切換到 **trellis** 資料夾，執行 **sudo ./trellis.py** 即可以運行起 **mininet** 腳本，並且透過 **ONOS** GUI 可以觀察到大量的 **Switch/Host** 存在。
- 透過下列指令三個指令都可以成功運行並且有回應 (須等待 dhcp 拿到 IP)
```
mininet$: h1 ping h4
mininet$: h1 ping 10.0.99.2
mininet$: h1v6 ping6 2000::9902
```
- 如果要結束整個模擬環境，可以透過下列指令將所有相關的 Process/Daemon 給移除
``` sh
sudo killall -9 dhclient dhcpd zebra bgpd
sudo mn -c
```

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