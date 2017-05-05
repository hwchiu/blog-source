---
layout: post
title: 'xml初體驗'
date: 2013-03-30 12:43
comments: true
categories: [C#]
tags:
	- C#
---

雖然現在已經有JSON的出現，更輕量級的資料表達方式
但不少系統依然使用者XML做為資料格式，譬如MSN的歷史訊息

這邊就來研究一下在C#中，如何對XML資料表進行操縱

首先要先知道，XML的資料表中，是以Node為單位，一層一層的往下增加資料，可以想像成一棵樹，必定有一個root，舉例來說
``` XML
	<Students>
	    <Student ID="1">
	       <Name>hwchiu</Name>
	       <Age>25</Age>
	    </Student>
	    <Student ID="2">
	       <Name>sppsorrg</Name>
	       <Age>55</Age>
	    </Student>
	</Students>
```


<!--more-->


Studnets就是所謂的root,底下有兩個node，都是student,然後每個node本身有自己的屬性(ID)，同時底下又有其他的node,這些node又有自己的屬性與對應的值。

	簡單介紹XML後，就來用用看C#中如何使用XML

C#中，關於XML有關的函式都在System.Xml中

##創立一個XML檔案
XmlDocument 是 XML的文件類別，要存取一個XML文件，就要透過這個類別
XmlDeclaration 是XML文件類別的開頭宣告，可以利用此函式為XML文件寫下其聲明
XmlElement 是XML文件中重要的一個物件，代表每一個節點，所有資料的搬移讀寫都靠操作節點完成
可以透過XmlElement::SetAttribute其屬性與其值，再將該XmlElement透過相關function來根據情況插入node

+ AppendChild 
+ PrependChild  
+ ReplaceChild 
+ InsertAfter 
+ InsertBefore


在進行一系列操作後，要執行XmlDocument::Save，這樣就完成寫入一個XML格式的檔案。
以下就來一個簡單範例

 

        public CreateXML()
        {
            InitializeComponent();
            XmlDocument xmlDocument = new XmlDocument();
            XmlDeclaration xmlDeclaration = xmlDocument.CreateXmlDeclaration("1.0", "", "yes");
            xmlDocument.PrependChild(xmlDeclaration);
            XmlElement nodeElement = xmlDocument.CreateElement("Students");
            xmlDocument.AppendChild(nodeElement);

            AddStudent(xmlDocument, 0156521, "hwchiu",22, "sppsorrg11.csg01@nctu.edu.tw");
            AddStudent(xmlDocument, 9717164, "sppsorrg", 18, "sppsorrg11.cs97@nctu.edu.tw");
            xmlDocument.Save("student.xml");
        }
        private void AddStudent(XmlDocument doc, int StudnetID, string name, int age, string email)
        {
            XmlElement elemEmployee = doc.CreateElement("Student");
            elemEmployee.SetAttribute("StudentID", StudnetID.ToString());
            doc.DocumentElement.AppendChild(elemEmployee);
            addTextElement(doc, elemEmployee, "name", name);
            addTextElement(doc, elemEmployee, "age", age.ToString());
            addTextElement(doc, elemEmployee, "email", email);
            
        }
        private void AddTextElement(XmlDocument doc, XmlElement nodeParent, string Tag, String Value)
        {
            XmlElement nodeElem = doc.CreateElement(Tag);
            nodeElem.AppendChild(doc.CreateTextNode(Value));
            nodeParent.AppendChild(nodeElem);
        }

***

	<?xml version="1.0" standalone="yes"?>
	<Students>
	  <Student StudentID="156521">
	    <name>hwchiu</name>
	    <age>22</age>
	    <email>sppsorrg11.csg01@nctu.edu.tw</email>
	  </Student>
	  <Student StudentID="9717164">
	    <name>sppsorrg</name>
	    <age>18</age>
	    <email>sppsorrg11.cs97@nctu.edu.tw</email>
	  </Student>
	</Students>
##讀取XML
###XMLReader  
使用XmlReader來逐步讀取XML，對於過大的XML檔案時，不會一口氣全部讀進memory
由於有實作IDisposable,所以使用using來自動釋放資源
reader會逐步讀取，每個tag分成 Element,Text,EndElement，以上述為例

>  <name> =>  Element;
>  hwchiu =>  Text;
>  </name>=> EndElement;

其中對於某些node中，有其attribute的，可以使用MoveToAttribute(`string name`)或是
MoveToNextAttribute()來遍尋，範例如下
``` c#
	using(XmlReader reader = XmlReader.Create("student.xml")){
	    while (reader.Read())
	    {
	        switch (reader.NodeType)
	        { 
                case XmlNodeType.Element:
                    Console.Write("<"+reader.Name);
                    while (reader.MoveToNextAttribute())
                        Console.Write(" " + reader.Name + "=" + reader.Value);
                    Console.WriteLine(">");
                    break;
                case XmlNodeType.Text:
                    Console.WriteLine(reader.Value);
                    break;
                case XmlNodeType.EndElement:
                    Console.Write("<" + reader.Name);
                    Console.WriteLine(">");
	        }
	    }
	}


```
Output

	<Students>
	<Student StudentID=156521>
	<name>
	hwchiu
	</name>
	<age>
	22
	</age>
	<email>
	sppsorrg11.csg01@nctu.edu.tw
	</email>
	</Student>
	<Student StudentID=9717164>
	<name>
	sppsorrg
	</name>
	<age>
	18
	</age>
	<email>
	sppsorrg11.cs97@nctu.edu.tw
	</email>
	</Student>
	</Students>

###XMLDocument
XMLDocument載入xml檔案時，會一次讀完，所以可以直接使用其方法來訪問各個節點
一樣以剛剛的student.xml為範例
先利用GetElementsByTagName取得所有node的list 集合
由於底下的`<name><age><email>`都是其child node,所以必須又要在取得一次NodeList,
如此反覆就可以取得所有資料

``` c#
	XmlDocument xml = new XmlDocument();
	xml.Load("student.xml");
	XmlNodeList nodeList = xml.GetElementsByTagName("Student");
	foreach (XmlNode parentNode in nodeList)
	{
	    if (parentNode is XmlElement)
	    {
	        XmlElement element = (XmlElement)parentNode;
	        String id = element.GetAttribute("StudentID");
	        XmlNodeList childList = element.ChildNodes;
	
	        Console.WriteLine("StudentID="+id);
	        foreach (XmlNode childNode in childList)
	        {
	            Console.Write("<" + ((XmlElement)childNode).Name + "> ");
	            Console.WriteLine(((XmlElement)childNode).ChildNodes.Item(0).Value);
	        }
	    }
	}
```
***
Output  

	StudentID=156521
	<name> hwchiu
	<age> 22
	<email> sppsorrg11.csg01@nctu.edu.tw
	StudentID=9717164
	<name> sppsorrg
	<age> 18
	<email> sppsorrg11.cs97@nctu.edu.tw
###LINQ
這邊使用LINQ來搜尋XML文件，所以必須要先使用  
**using system.xml.Linq** 

為了讓Linq能夠順利運行，這邊使用的物件是XElement以及XNode
首先以XElement的方式來讀取檔案，接者使用linq的語法從中獲取我們想要的資訊
範例中先以root.Elements("Student")獲取所有Student的節點，接者在去比較其屬性中的
StudentID，來得到特定的資訊。

最後回傳的資訊是個`IEnumerable<XElement>`的型態，使用foreach來拜訪
這邊的XElement本身的值就是
	
	<Student StudentID="156521">
	  <name>hwchiu</name>
	  <age>22</age>
	  <email>sppsorrg11.csg01@nctu.edu.tw</email>
	</Student>

所以為了獲得其中的資訊，就必須要繼續拆解該節點，繼續以foreach的方式取得
每個XNode都代表者一行資訊如
> `<name>hwchiu</name>`  

此時可以將XNode給轉型為XElement，就可以利用`<Name/Value>`的方式分別取得
`<name>跟hwchiu`  

若只是想要取得特別的資訊，可以直接透過Linq的語法來查詢，寫起來會更加簡潔及可讀。
```c#
	XElement root = XElement.Load("student.xml");
    var student =
        from el in root.Elements("Student")
        where (string)el.Attribute("StudentID") == "156521"
        select el;
    foreach (XElement el in student)
    {
        foreach (XNode node in el.Nodes())
        {
            Console.WriteLine(node);
        }
    }
```

***
Output  

	<name>hwchiu</name>
	<age>22</age>
	<email>sppsorrg11.csg01@nctu.edu.tw</email>
