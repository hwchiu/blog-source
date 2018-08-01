---
layout: post
title: nmap
date: '2013-06-15 09:25'
comments: true
tags:
  - System
abbrlink: 16805
---


nmap是一個linux下的工具

nmap - Network exploration tool and security / port scanner
<!--more-->


這邊記錄一下nmap的用法

nmap -sP 140.113.214.79/27  

-sP: Ping Scan - go no further than determining if host is online
用ping去掃目標內的所有IP，並顯示有回應的IP，所以若對方是windows7且沒有打開ping的回應，則也會被當作host down

nmap -sL 140.113.214.79/27    

-sL: List Scan - simply list targets to scan
只是單純的列出對方的hostname以及IP，不送出任何封包去檢測

nmap -O 140.113.214.94
nmap -A 140.113.214.94

-O: Enable OS detection
-A: Enables OS detection and Version detection, Script scanning and Traceroute

掃描對方主機的OS系統

nmap -PS/PA/PU/PY[portlist] 140.113.214.94

-PS/PA/PU/PY[portlist]: TCP SYN/ACK, UDP or SCTP discovery to given ports

用不同的方式去掃描特定的PORT。

- PS 用TCP 搭配 SYN FLAG去偵測。
- PA 用TCP 搭配 ACK FLAG去偵測。
- PU 用UDP去偵測。
- PY 用SCTP去偵測。


nmap -sS/sT/sU 140.113.214.94

採用不同的方式去掃描所有port。

- sS (TCP SYN scan) .
- sT (TCP connect scan)
- sU (UDP)

nmap -v 140.113.214.94
顯示出詳細一點的資訊

