---
layout: post
title: 'ZNC IRC bouncer'
date: 2014-07-23 02:22
comments: true
tags:
	- FreeBSD
	- IRC
---
Environment
-----------
- FreeBSD 10.0-Release


INSTALL
-------
####pkgng
1. pkg install znc

####Ports
1. cd /usr/ports/irc/znc 
2. make config
3. make install & clean

<!--more-->


Config
------
####znc --makeconf
- add listen port.
- add user
- add network. ex: freenode
	-	add irc server. ex: irc.freenode.net
	- you can add the default channel passowd  by a key=? option in znc.conf

Usage
-----
####Find a irc client
- [AndChat on Android](https://play.google.com/store/apps/details?id=net.andchat)
- [kiwiirc on Web](https://kiwiirc.com/)
