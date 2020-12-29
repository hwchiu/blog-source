---
layout: post
title: 'Bluez 5.x '
date: '2014-10-25 13:02'
comments: true
tags:
  - System
  - Bluetooth
  - Linux
abbrlink: 16746
---
Introduction
------------
從[官方網站](http://www.bluez.org/)的說明
>>>
BlueZ is official Linux Bluetooth protocol stack. It is an Open Source project distributed under GNU General Public License (GPL). BlueZ kernel is part of the official Linux kernel since version 2.4.6.

從這邊可以看得出來，Bluez是一套在linux系統專，專門負責bluetooth裝置連線的軟體，因此滿多linux-based的系統都會使用此套軟體作為與Bluetooth裝置連接的工具。

在版本方面，目前最新的版本是5.24(2014/10/25)。然而在Ubuntu 14.04的官方套件中，依然使用4.101的版本，這邊差了一個大的版本號。下列列舉一下 4.x與5.x版本的較大的差異性
- Interface的部分完全改掉，在5.x中已經沒有了AudioSink等Profile相關的interface
- 5.x中原生不再支援a2dp、hsp等profile，必需要依靠第三方套件支援。

    <!--more-->



Installation
------------
- 安裝部分就參考文件內的說明進行configure以及make、make install即可，可以根據configure的需求去調整。
- 由於a2dp部分bluez原生不再支援，這邊要使用第三方套件[PulseAudio](http://www.freedesktop.org/wiki/Software/PulseAudio/)來處理，注意的是要5.x版本後才有支援bluez5。因此這邊就到PulseAudio的官方網站去下載，記得在`configure`的部分要指定--enable-bluez5，這樣才會編譯出與bluez5相關的套件。
- 安裝過程可能會遇到一些lib缺少的問題，這邊就依照所缺少的去安裝即可。

Usage
-----
- run the pulseaudio  as daemon
- use the dbus-send command to connect the BT device
	- 這邊可以使用bluetoothctl指令來操作，包含與裝置的配對、連線等。
  - power on
  - agent on
  - default-agent
  - scan on
  - pair ###
  - connect ##
- 播放音樂方面，由於bluez 5.x沒有辦法支援alsa，因此沒有辦法透過mplay來播放，必須要透過pulseaudio來播放音樂
	- pacmd --help
