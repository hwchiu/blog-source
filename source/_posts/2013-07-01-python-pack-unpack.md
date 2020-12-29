---
layout: post
title: Python-pack_unpack
date: '2013-07-01 13:06'
comments: true
tags:
  - Python
abbrlink: 51029
---
pack & unpack

根據特定的格式來讀取或封裝資料。
<!--more-->

格式部份分成兩個部份

byte-order:
這邊可以決定採用big-endian 或是 little endian，
如果沒有給的話，預設是採用系統的方式去做，那這邊比較要注意到的是
以前再寫C語言的時候，都會有所謂的htons...類的轉換，在這邊可以使用'!'
這個符號來解決這個問題，他會自己使用network的Byte order rule去解讀資料，所以有在用網路連線傳資料的話，一定要用！
避免資料解讀錯誤的問題。

format-characters:

常用的有

- x:pad byte,就忽略他
- h:short,2 Byte
- s:char[], 代表字串，使用時前面要加上數字
- i:long int, 4Byte
- B:unsigned char, 1 Byte

詳細的格式資訊請參考官網
[Python struct](http://docs.python.org/2/library/struct.html "Python struct")

這邊來個簡單範例
假設今天我們撰寫屬於自己的網路遊戲
然後我們玩家每次上線時，SERVER都會傳送一份玩家的資料給Client

這份資料包含了
- 遊戲版本
- 玩家ID
- 玩家的座標(XY)
- 玩家當前的財產
- 玩家的職業
- 玩家的等級
- 玩家的經驗值


每個資料所需要的型態跟大小如下敘述

```
Myheader(){
  uint8:version
  uint8:playerID
  uint16:x
  uint16:y
  uint32:momey
  char10:profession
  uint8:level
  uint32:experience
}
```
所以傳送資料過來的時候，我們必須要謹慎的按照這個規格去放置我們的資料。
### Example ###
假設
- version = 1
- playerID = 56
- x = 123
- y = 2341
- momey = 5566217
- profession = "warrior"
- level = 128
- experience = 2147383611

```python
data = pack('2B2HI10sBI',version,playerID,x,y,momey,profession,level,experience)
//'\x018{\x00%\t\x00\x00\t\xefT\x00warrior\x00\x00\x00\x80\x00\xdb\xff\xff\x7f'
unpack('2B2HI10sBI',data)
(1, 56, 123, 2341, 5566217, 'warrior\x00\x00\x00', 128, 2147483611)

```
