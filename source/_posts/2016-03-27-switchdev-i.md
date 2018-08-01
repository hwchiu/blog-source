---
layout: post
title: Switchdev I
date: '2016-03-27 15:25'
comments: true
tags:
  - System
  - Kernel
  - Switchdev
  - Network
keywords: 'Linux,Kernel,Network,SwitchDev,HardwareSwitch'
abbrlink: 14196
---
Introduction
------------
- Switchdev 在 linux kernel 3.19+ 以後才正式支援的，此專案希望能夠讓整合 hardware switches 與 Linux kernel。
- 以前的 hardware switch 都有實作自己的 L2/L3 offloading，同時廠商會在 user space 提供自己的 tool 用來操控該 switchdev。
- 在此架構下，很多常用的 user space tool，如 ethtool, ip, brctl ..等都沒有辦法針對 hardware swtich 去控制，這會使得上層的軟體都要針對不同的底層硬體去客製化處理
- 為了解決這個問題，希望在 kernel 中加入一層 switchdev，各廠商在 kernel 內實現自己 driver 的 switchdev，然後 swtichdev 本身會與原本的 user space tool 整合，這樣的話 user space 就不用額外提供 tool 了。

    <!--more-->


以下使用 [Hardware switches - the open-source approach](http://people.netfilter.org/pablo/netdev0.1/slides/pirko-switchdev-slides.pdf) 內的圖片來說明
Before Switchdev
----------------
![](https://lh3.googleusercontent.com/-cj2IIISQSBk/Vvf_UAiHg9I/AAAAAAAAFME/QxCE0N_zMAMMLDssB0MbUZHZbNBLlMNNQCCo/s852-Ic42/beforeSwitch.PNG)

- 此圖片顯示的就是目前的狀態，右邊顯示的是一般常見的 kernel 狀況，包含一些 tool 與底層 NIC 是如何操作的。
- 左邊則是當前 hardware switch 的普遍設計，整個操作都跳過 linux kernel，一切都是廠商自己的程式在處理而已。
- 當前架構下，沒有辦法於系統上觀察到實體 switchdev 到底有哪些 port，就像現在多數的家用 router 一樣，明明有四個 lan 孔，但是透過 ifconfig 看都只會有一個。

After Switchdev
---------------
![](https://lh3.googleusercontent.com/-x9BeDYWCBxA/Vvf_UPXkwiI/AAAAAAAAFMI/UdFkhv7AqrUm4yTVvF0AEmdbgGvOdTIbwCCo/s1033-Ic42/AfterSwitch.PNG)
- 此圖顯示的是 switchdev 此專案希望的架構
- 此架構中，廠商根據 switchdev 定義好的架構去實現自己的 driver，這樣原生的 tools 都可以直接對真正的 hardware switch 進行操作，廠商也不需要自己在額外開發 user space tool 了。

Vendor Implementation
---------------------
- 第一個實作完成的 switch driver 是 [Rocker](http://people.netfilter.org/pablo/netdev0.1/papers/Rocker-switchdev-prototyping-vehicle.pdf)
- Rocker 的 code 可以在 kernel 內的 **/drivers/net/ethernet/rocker/** 內看到
- Mellanox 的 code 可以在 kernel 內的 **/drivers/net/ethernet/mellanox/mlxsw/** 內看到

Next Page
---------
接下來的章節將會介紹 switchdev 在 kernel 內的架構實作以及與 Rocker 這個 switch driver 是如何互動的。
大抵架構如下
![](https://lh3.googleusercontent.com/-7BWbz43luZw/VwO0afnRVaI/AAAAAAAAFNk/Psakav4xP3MX-8b9N6h_0nAzSllHjkgEQCCo/s824-Ic42/SwitchDev.png)




Reference
---------
- [kernel document](https://www.kernel.org/doc/Documentation/networking/switchdev.txt)
- [Hardware switches - the open-source approach](http://people.netfilter.org/pablo/netdev0.1/slides/pirko-switchdev-slides.pdf)
- [Rocker switchdev prototyping vehicle](http://people.netfilter.org/pablo/netdev0.1/papers/Rocker-switchdev-prototyping-vehicle.pdf)
