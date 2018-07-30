---
title: Anki 使用感想 (tutorial)
keywords: 'Anki, System, AddToAnki'
tags:
  - System
  - Anki
abbrlink: 34221
date: 2017-08-05 16:05:07
description:
---

Anki 使用感想 (後)
---------------

有鑑於之前的文章 [Anki 使用感想](https://blog.hwchiu.com/2017-03-01-anki-thoughts.html) 所提到的自動爬網站並加入卡片的功能對於一般使用者不方便理解與使用。

但是目前也時間沒有什麼多的時間去將其改善成一個友善的*one-click*的程式來完成這件事情，所以決定先寫一篇文章來手把手教學如何在 **Windows** 上面設定這個程式。等之後有時間與想法可以再來將其整個改善。
畢竟此程式一開始就只是為了讓自己方便使用而已，所以在程式撰寫上各種醜陋XD


## Dict
本程式主要主要的字典查詢網站是透過 **Yahoo Dict** 為主，以 **infest** 當作[範例](https://tw.dictionary.search.yahoo.com/search?p=infest&fr2=dict)。
目前抓取的邏輯非常簡單!
卡片的正面是該單字的英文與其英標，而其背面則是剩下的所有解釋。
如下圖所示
![](http://i.imgur.com/jfEb7u0.png)

如果對於這邊卡片的正反面，字典抓取來源網站有任何想法的都可以提出來。


<!--more-->

## Environment
要執行這個程式前，必須有一些軟體要先安裝在電腦內，包含了
1. Anki Application
2. Anki Source Code + addtoAnki Source Code
3. Python3

這邊接下來會針對這三個進行說明，並且解釋每個部分有哪邊需要注意。

### Anki Application
基本上一般的使用者基本上都已經會將此程式給安裝完畢，不然系統上也沒有辦法運行 Anki 來使用。
這邊要注意的是，接下來的程式會需要用到兩個設定，分別是 **個人檔案** 以及 **牌組**。

因為本程式會自動將爬完字典網站的結果寫入到特定**個人檔案**的特定**牌組**內，所以必須要取得系統上關於這兩個資訊所對應的**檔案路徑**。

這兩個設定可以參考下列如圖來瞭解分別是什麼(請原諒我如此潦草的作畫..)
圖中紅色框起來的地方就是**個人檔案**的名稱，而藍色框框則代表的是**牌組**的名稱。
![](http://i.imgur.com/5pcgQWu.png)


假設我想要透過此程式自動加入到個人檔案(hwchiu)的牌組(Test)，則首先到檔案總管中輸入 **%userprofile%\AppData\Roaming\Anki2**
這時候畫面應該會出現如下圖的訊息
![](http://i.imgur.com/24OQOZX.png)

則時候滑鼠點選進去到 **hwchiu**裡面，又會看到如下圖
![](http://i.imgur.com/b6YnnjN.png)
這時候該資料夾的路徑是 
**C:/Users/hwchiu/AppData/Roaming/Anki2/hwchiu/collection.anki2**
這個路徑必須要記錄下來，之後我們的程式會需要這個路徑來對底下的 **collection.anki2** 這個 **collection** 來處理。

這樣我們就已經處理好了 **個人檔案** 所需要的檔案路徑了，至於 **牌組**的部分稍後在城市內處理即可。

### Python3
由於本程式是透過 Python3 撰寫，所以請先在系統上安裝 Python3，目前測試過
**python-3.6.2.exe** 是可以的，不過我相信只要是 Python3 應該都沒有問題。
Python3 可以從[官網](https://www.python.org/downloads/)這邊下載。
安裝的時候請特別注意該 Python 的安裝路徑，如下圖
![](http://i.imgur.com/M08pRnQ.png)
本範例的安裝路徑是，請記住自己的安裝路徑，此外，如果你有客製化自己的安裝內容的話，請確保 **pip** 一定要安裝。
```
C:\Users\hwchiu\AppData\Local\Programs\Python\Python36-32\
```

### Anki Source Code + addtoAnki Source Code
首先，以下程式放置的位置隨個人喜好，自己找得到就好。我則以 **D:\add_anki**為範例。
首先下載上述兩個檔案，分別是
[AddToAnki Source Code](https://github.com/hwchiu/addToAnki/archive/master.zip)
[Anki Source Code](https://github.com/dae/anki/archive/master.zip)
然後將其放到 **D:\add_anki**裡面，然後分別解壓縮。
這時候該資料夾看起來就像
![](http://i.imgur.com/y8eNZKK.png)

這邊我們要記住的一個路徑是 **D:\add_anki\anki-master**，因為我們接下來要讓 **python** 知道 **anki** 的原始碼在哪裡，所以需要此路徑。

到這一步驟為止，相關的檔案都已經抓取下來了，接下來開始要針對一些部分進行調整，這部分會比較瑣碎一點，主要是我的程式什麼都沒有自動判斷XD，因此有一些東西要自己去處理。

## Configuration
接下來大概有三個步驟要完成
1. 設定系統環境變數，主要會設定 **PATH**，**PYTHONPATH** 以及 **PYTHONIOENCODING**。
2. 透過 **PIP** 安裝第三方套件供 PYTHON 使用
3. 設定本程式要執行的相關設定，如前述提到的**個人檔案**路徑位置以及對應的**牌組**


### Environment Variable
不論是 windows7或是windows 10，設定環境變數的方法大同小異，不熟悉的人可以參考[win 7 環境變數設定](http://huangjung1216.pixnet.net/blog/post/148662170-win-7-%E7%92%B0%E5%A2%83%E8%AE%8A%E6%95%B8%E8%A8%AD%E5%AE%9A) 這網站來
設定
首先，為了讓我們能夠在 **CMD** 的環境中能夠直接執行 **python** 以及 **pip**，我們要先針對 **PATH** 變數去設定。

#### PATH
基本上此變數 **PATH** 都已經存在於系統之中，所以我們只要擴充其數值就好。
這邊需要上述安裝 **PYTHON3** 時所提到的安裝路徑，
```
C:\Users\hwchiu\AppData\Local\Programs\Python\Python36-32\
```
我們需要幫 PATH 加入額外兩個變數，分別是
```
C:\Users\hwchiu\AppData\Local\Programs\Python\Python36-32\
C:\Users\hwchiu\AppData\Local\Programs\Python\Python36-32\scripts
```
我這邊以我的 **windows10**為範例，大概會如下圖這樣
![](http://i.imgur.com/6UKbOi8.png)

#### PYTHONIOENCODING
接下來就如同上面的步驟設定
**PYTHONIOENCODING**其數值為 **utf8**
![](http://i.imgur.com/6bXxreB.png)

#### PATHONPATH
**PATHONPATH**其數值為 **%PYTHONPATH%** 以及 **D:\add_anki\anki-master**。
![](http://i.imgur.com/FkY5BPB.png)

按下儲存都完畢後，要如何檢查自己的環境變數是否有設定成功，請打開 CMD(命令提示字元)，要如何打開請自行 google
打開後執行 `set` 這個指令就會在畫面上顯示當前所有的環境變數，這時候再去檢查剛剛設定的是否都有存在。
![](http://i.imgur.com/AwlTQS5.png)
### pip
假設上面的設定都正常的話，在 CMD 下面執行 pip，應該會出現類似下面的文字 (版本號可能會些許不同)
```
C:\Users\hwchiu>pip --version
pip 9.0.1 from c:\users\hwchiu\appdata\local\programs\python\python36-32\lib\site-packages (python 3.6)
```
這時候依序輸入下列四行指令
```
pip install six
pip install packaging
pip install appdirs
pip install bs4
pip install lxml
pip install decorator
pip install pyaudio
```

### Config
接下來我們將資料夾切換到 **D:\add_anki**(這邊的路徑是上述第三步驟所下載的位置)，然後透過滑鼠的點擊，直接進入到 **D:\add_anki\addToAnki-master\examples\YahooDict** 裡面。
可以看到裡面目前有四個檔案，分別是
- config.json
- input
- main.py
- README.md
這邊要注意的檔案只有兩個，一個是 **config.json** 以及 **input**。
用任何你喜歡的文字編輯器打開 **config.json**,其內容大概如下
```json
{
    "profiles": [
 	 	{
	        "file":"input",
	        "deck":"test",
	        "collection":"C:/Users/hwchiu/AppData/Roaming/Anki2/hwchiu/collection.anki2"
	   	}
    ]
}
```
這邊有三個資訊需要你自己去修改，分別是 **file**, **deck** 以及 **collection**。
首先 **collection** 就是之前提過的**個人檔案的位置**，而 **deck** 則是你想要加入到的**牌組名稱**，而 **file** 則是指名你英文單字的來源檔案(該檔案的格式是每個單字一行，可以參考本來的 input)
所以按照我目前預設的設定，其意義就是將 **input** 檔案內的英文單字都去自動去 Yahoo Dict 抓取解釋並且產生卡片，這些卡片都加入到 **hwchiu** 這個個人檔案的 **test** 牌組中。
當這些設定都完畢後，就可以來準備測試了。

## Run
1. 強烈建議一開始先創立一個測試用的牌組來測試，確認一切都沒有問題後，才將 deck 改成自己常用的 deck
2. 執行前請關掉正在運行的 **Anki**，因為同時只能有一隻程式去存取 **Anki** 的資料庫。否則可能會看到類似下面的訊息
```
sqlite3.OperationalError: database is locked.
```
3. 直接點擊 **main.py** 即可。
運行的畫面大致如下
![](http://i.imgur.com/hKMLXPW.png)
4. 運行完畢後就可以打開 Anki，確認卡片都有被加入進去。

## Summary
目前的程式建置上還頗麻煩的，不過只要建置完畢後，之後都只需要修改來源檔案的內容，然後執行 **main.py** 就可以發呆等程式跑完即可。
至於之後是否將這些東西更簡單地去處理，我想之後有時間&有需求的時候再來考慮好了。

### Reference
- [HuangJung1216的部落格](http://huangjung1216.pixnet.net/blog)
