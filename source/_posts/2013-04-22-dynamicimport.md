---
layout: post
title: Python 動態載入模組
date: '2013-04-22 11:59'
comments: true
tags:
  - Python
abbrlink: 39552
---



最近在弄irc機器人，希望這個機器人能夠靈活一些，因此把所有功能都弄成module

機器人在掛上這些module來完成各種能力，心中的設想架構如下

	--------ircbot
		|-------config.json
		|
		|
		|-------server.py
		|
		|
		|-------modules
			|
			|-----googleSearch
			|   |
			|   |---googleSearch.py
			|
			|-----wikiSearch
			|   |
			|	|---wikiSearch.py
			|
			|-----echoServer
				|
			    |---echoServer.py

<!--more-->

config.json 是主要的設定檔，其中包含了要載入哪些module

server.py是主要的server程式，可以透過指令重新去載入config.json

當server載入設定檔後，會動態的去把modules資料夾底下的py都載入，這樣未來當有新功能要增加的時候，只要修改config.json，然後發送指令

叫server重新載入就可以獲得新功能，而不需要整個server重開。

參考[Telling __import__ where to look - am I stuck with sys.path?](http://stackoverflow.com/questions/7218673/telling-import-where-to-look-am-i-stuck-with-sys-path)
以及[python3.0中重载模块](http://eriol.iteye.com/blog/1113588)後

整理如下

首先要先創造出一個pseudo-package，sys.modules是一個dict的物件，由module name mapping 到 對應的module

這邊先創造一個tuple，key='plugins' value = type為sys的物件，叫做plugins

接下來把我們要載入的module路徑都加入道剛剛創立的物件之中，

利用__path__這個串列，把路徑一一加入進去

這樣在pseudo-package  plugins底下，已經看得到我們的module了!

最後再利用importlib.import_module('plugins.'+moduleNmae) 來把這些module全都載入

值得注意的是，由於module被載入一次後，即使你修改了code,利用這個方法重新載入，依然沒有辦法改變其行為

所以必須要使用reload這個function重新載入值，並讀取新的內容，因為module載入一個很大開銷的動作，因為每次都要尋找文件、編譯成

bytecode、轉成執行碼，因此這個行為必須要透過reload強制重做才可以達成。

最後就可以呼叫每個module的方法來達成動態載入的功能了

範例如下

```

    def dynamicLoadModules(self):
        sys.modules['plugins'] = self.plugins = type(sys)('plugins')
        self.plugins.__path__ = []
        for path in self.config['MODULES']:
            path = os.path.join(sys.path[0],'modules',path)
            self.plugins.__path__.append(path)

 		##dynamic load modules
        self.modules = []
        self.modules = [ importlib.import_module('plugins.'+module) for module in self.config['MODULES']]
        for module in self.modules: #用此來重新載入module http://eriol.iteye.com/blog/1113588
            reload(module)
		for module in self.modules:
			module.run()

```
