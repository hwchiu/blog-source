---
layout: post
title: 'Python-translate'
date: 2013-06-13 21:30
comments: true
tags:
	- Python
---

Python中有個很強大的字串轉換工具 maketrans 跟 translate


	str.translate(table[, deletechars]);
	Parameters
	table -- You can use the maketrans() helper function in the string module to create a translation table.
	
	deletechars -- The list of characters to be removed from the source string.

字串中只要有符合deletechars中的字元都會被刪除，然後剩下的字元就會依照table裡面的mapping來做轉換。

這個mapping的就要利用string.maketrans()來幫忙產生囉,

	str.maketrans(intab, outtab]);
	Parameters
	intab -- This is the string having actual characters.
	
	outtab -- This is the string having corresponding mapping character.

intab跟outtab兩者的長度必須要一樣，會把intab中每一個字元與outtab中相同位置的字元做mapping。


舉例來說

<!--more-->

``` python
	intab = "aeiou"
	outtab = "12345"
	trantab = maketrans(intab, outtab)
```	
	

就會產生一個mapping,把aeiou分別轉換成12345。
``` python
	input="abcdefgh"
	input = input.translate(trantab)
```
input就會變成 "1bcd2fgh"

那如果改成
``` python
	input="abcdefgh"
	input = input.translate(trantab,"fgh")
```
input就會變成 "1bcd2"

再來個簡單範例，希望能夠把所有的小寫轉成大寫，並把非英文字母外的所有字元都給刪除掉。

``` python
import string
```



#取得所有英文大小寫的集合
``` python
	lower = ''.join(map(chr,range(97,123)))
	upper = lower.upper()
```
#創立一個對照表，可以把所有小寫轉成大寫
``` python
	ltu = string.maketrans(lower,upper)
```	
#接下來要利用捕集的方式取得非英文字母以外的所有字元，因此就用所有字元-英文字母
#創立一個代表所有字元的字元表
``` python
	allchars = string.maketrans('','')
```
#利用translate的方式，取得所有非英文字母的集合
``` python
	delete = allchars.translate(allchars,lower+upper)
```	
#定義一個對應的function,傳入的字串利用ltu跟delete，就能夠把所有非英文字母都刪除，並且小寫轉大寫了。
``` python
	def makefilter(input):
	    print input.translate(ltu,delete)
```	


makefilter("asd@*(#)@*!)#1231321")
ASD
makefilter("cxczzcjklk21l3j@!#@!#111")
CXCZZCJKLKLJ

