---
title: 'C#中以ZedGraph畫統計圖'
keywords: 'C#, sharp, zedgraph, graph'
date: '2013-04-14 17:01'
comments: true
tags:
  - 'C#'
abbrlink: 19231
description: 在C#中繪製統計圖表，如折線圖、圓餅圖、長條圖 ,除了可以使用內建的Graphics物件外，還可以使用第三方的套件來畫圖,這邊就介紹第三方套件 ZedGraph，下次再介紹以內建的方法來繪圖

---


# INSTALL
目前最新的版本是 [5.16](http://nuget.org/packages/ZedGraph)
安裝方法請使用 [Package Manager Console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console)來安裝，

>PM> Install-Package ZedGraph

之後就會幫你安裝完成，此時你的資料夾底下會有個packages的資料夾，裡面就放有此套件的dll。


# USAGE
ZedGraph本身有提供自定義的使用者元件，所有的繪圖都是在該元件上運作，所以必須要先加載該元件
元件=>選擇項目=>瀏覽=>pkakage/zedgraph.dll

順利的話，就可以直接在工具箱中拖曳該元件囉

使用的方法，[官方網站](http://www.codeproject.com/Articles/5431/A-flexible-charting-library-for-NET)(有點舊)上面有許多的教學跟範例
仔細研讀後就大概會用了

以官方範例畫一張曲線圖來說明

``` c#

	private void CreateGraph( ZedGraphControl zgc )
	{
	   // get a reference to the GraphPane
	   GraphPane myPane = zgc.GraphPane;

	   // Set the Titles
	   myPane.Title.Text = "My Test Graph\n(For CodeProject Sample)";
	   myPane.XAxis.Title.Text = "My X Axis";
	   myPane.YAxis.Title.Text = "My Y Axis";

	   // Make up some data arrays based on the Sine function
	   double x, y1, y2;
	   PointPairList list1 = new PointPairList();
	   PointPairList list2 = new PointPairList();
	   for ( int i = 0; i < 36; i++ )
	   {
	      x = (double)i + 5;
	      y1 = 1.5 + Math.Sin( (double)i * 0.2 );
	      y2 = 3.0 * ( 1.5 + Math.Sin( (double)i * 0.2 ) );
	      list1.Add( x, y1 );
	      list2.Add( x, y2 );
	   }

	   // Generate a red curve with diamond
	   // symbols, and "Porsche" in the legend
	   LineItem myCurve = myPane.AddCurve( "Porsche",
	         list1, Color.Red, SymbolType.Diamond );

	   // Generate a blue curve with circle
	   // symbols, and "Piper" in the legend
	   LineItem myCurve2 = myPane.AddCurve( "Piper",
	         list2, Color.Blue, SymbolType.Circle );

	   // Tell ZedGraph to refigure the
	   // axes since the data have changed
	   zgc.AxisChange();
	}
```

該第三方套件提供的自定義元件叫做**ZedGraphControl**
而我們要畫圖則是在**GraphPane**上去操作

所以首先要先從取得GraphPane

接者先定義該圖表的

**標題**，**橫軸標題**，**縱軸標題**

接者我們要加入資料到該圖表中，對於一張2D的圖表來說

使用**PointPairList**這個物件來存放XY的座標位置

最後使用**LineItem**來把相關的點給連起來


由於資料加入圖表中，我們的座標軸要根據資料產生變化，此時就呼叫
AxisChange()來改變。



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