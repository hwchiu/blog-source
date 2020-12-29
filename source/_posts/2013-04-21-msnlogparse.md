---
layout: post
title: 'MSN LOG解析以C#'
date: '2013-04-21 20:11'
comments: true
tags:
  - 'C#'
abbrlink: 28275
---


Msn的log採用的格式是XML，隨便打開一個LOG後仔細檢視，可以發現msn log的訊息格式大概是採這樣

	<Message Date="2012/3/23" Time="下午 11:33:12" DateTime="2012-03-23T15:33:12.790Z" SessionID="1">
		<From><User FriendlyName="邱 渣"/></From>
		<To><User FriendlyName="XXX"/></To>
		<Text>明天會到否</Text>
	</Message>

<!--more-->


每一則訊息，本身的屬性會包含該訊息的發送時間 ，有兩種格式，後面的790Z就不清楚是什麼意思了，SessionID這個屬性

也不是很清楚，但是這些都不重要

利用Date跟Time就可以取得基本時間了。

接者可以看到底下有三個屬性，代表訊息發送者，訊息接收者，以及發送的訊息為何

如果有啟動顏色跟字型的話，TEXT欄位就會變成下列樣子，會有屬性標示其顏色與字型


	<Text Style="font-family:Microsoft JhengHei; color:#000000; "> test </Text>


在C#中，我這次使用XmlElement來做為解析XML的工具，載入檔案後，因為我們只關心訊息的傳送，

所以先利用GetElementsByTagName("message")來取得所有Message有關的nodes

接者針對這個結點內的所有資料去進行資料抓取，我們的目標有 時間、發送者、傳送文字

先將XmlNode轉型為XmlElement的類型，這樣方便處理，然後利用GetAttribute來取得Message的屬性

我們就可以知道每個對話的Date跟Time。接者要存取其child(From,To,Text)這些的值

這邊比較要注意的是這兩種的差別


	<From>"邱渣"</From>


	<From><User FriendlyName="邱渣"/></From>



以Type1來說，邱渣是From這個結點的值，可以利用childList[0].value 取得發送者的名稱

但是對Type2來說，邱渣是From這個結點底下的一個結點中的屬性，所以就要利用childList[0].FirstChild

的方式來取得<User>這個結點，再搭配Attributes[0].Value來取得第一個屬性的值，如此才可以取得"邱渣"的值

所以利用childList[2].FirstChild.Attributeds[0].Value就可以取得文字訊息了!

另外，如果要取得文字的顏色跟字型的話，利用

childList[2].GetAttribute("Style")

接者在去自己處理字串來取得字型跟顏色。

範例code如下

```
	xml = new XmlDocument();
	xml.Load(filename);
	XmlNodeList nodeList = xml.GetElementsByTagName("Message");
	foreach (XmlNode parentNode in nodeList)
	{

		XmlElement element = (XmlElement)parentNode;
		string Date = element.GetAttribute("Date");
		string Time = element.GetAttribute("Time");
		XmlNodeList childList = element.ChildNodes;
		data += childList[0].FirstChild.Attributes[0].Value + " 說 (" + Time + ")\r\n";
	}
```
