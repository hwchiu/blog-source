---
title: 'Json in C-Sharp(C#)'
keywords: "csharp,json,example"
date: '2013-04-06 22:49'
comments: true
tags:
  - CSharp
abbrlink: 3890
description: 這篇文章用來介紹在 C# 中如果想要針對 json 的種格式的資料進行處理的話，可以使用那些函式庫來幫忙處理。同時也針對不同種的使用方法給予相對應的範例程式碼，讓大家可以更快速且方便地用 C# 來操作 json 的格式資料。

---

# Preface
JSON相對於XML來說較為簡單，沒有繁瑣的標籤，取而代之的就是一對<key,value>
如
```json=
{
	"name":
	{
		"firstName": "Hung-Wei",
		"lastName":  "Chiu"
	},
	"birthday":  "19900317",
	"studentID": "0156521",
	"email":
	[
		"sppsorrg@gmail.com"
		"hwchiu@cs.nctu.edu.tw"
	]

}
```

# CSharp
本文將介紹如何以C#來處理JSON格式的物件，
所以這邊推薦使用**JSON.NET**這個被廣泛使用的函式庫

# 安裝
請參考官方網站說明 [JSON.NET](http://json.codeplex.com//)

# JSON.NET使用
JSON.NET中，對於JSON的操作，主要分成兩大類

- Serializing and Deserializing JSON
	主要是針對物件使用的，能夠將JSON的資料跟物件快速的轉移
	缺點就是對應每個JSON資料，都要創立對應的物件來存取
- LINQ to JSON
	第二種就是利用JSON Object的方式來操作JSON，使用起來比較直覺也比較有彈性。

以下就針對使用這兩種方法來實際操作json
同時就以上述的範例作為json data

## Create JSON Format Data

## Serializing
``` c#=
	public class Student
	{
	    public Dictionary<string,string> name {get;set;}
	    public string birthday { get; set; }
	    public string studentID { get; set; }
	    public List<string> email {get;set;}

	}
    Student student = new Student
    {
       name = new Dictionary<string,string>
       {
           {"firstName","Hung-Wei"},
           {"lastName","Chiu"}
       },
       birthday = "19900317",
       studentID = "0156521",
       email = new List<string>
       {
        "sppsorrg@gmail.com",
        "hwchiu@cs.nctu.edu.tw"
       }
    };

    string a = JsonConvert.SerializeObject(student, Newtonsoft.Json.Formatting.Indented);
    Console.WriteLine(a);
```

```json=
	Output
		{
		  "name": {
		    "firstName": "Hung-Wei",
		    "lastName": "Chiu"
		  },
		  "birthday": "19900317",
		  "studentID": "0156521",
		  "email": [
		    "sppsorrg@gmail.com",
		    "hwchiu@cs.nctu.edu.tw"
		  ]
		}
```

## LINQ TO JSON
這種類型下，有非常多的方法可以使用

- JTokenWriter
- Anonymous Type
- Dynamic Object
- JObject and JProperty

這邊只介紹使用Anonymous Type的方式

```c#
	JObject o = JObject.FromObject(new
	{
	    name = new Dictionary<string, string>
	   {
	       {"firstName","Hung-Wei"},
	       {"lastName","Chiu"}
	   },
	    birthday = "19900317",
	    studentID = "0156521",
	    email = new List<string> {
	        "sppsorrg@gmail.com",
	        "hwchiu@cs.nctu.edu.tw"
	    }

	});
	Console.WriteLine(o.ToString());
```
	Output:
		{
		  "name": {
		    "firstName": "Hung-Wei",
		    "lastName": "Chiu"
		  },
		  "birthday": "19900317",
		  "studentID": "0156521",
		  "email": [
		    "sppsorrg@gmail.com",
		    "hwchiu@cs.nctu.edu.tw"
		  ]
		}
### Read JSON Format Data
### Serializing

``` c#
	string json = @"	{
	  'name': {
	    'firstName': 'Hung-Wei',
	    'lastName': 'Chiu'
	  },
	  'birthday': '19900317',
	  'studentID': '0156521',
	  'email': [
	    'sppsorrg@gmail.com',
	    'hwchiu@cs.nctu.edu.tw'
	  ]
	}";
	Student student = JsonConvert.DeserializeObject<Student>(json);
	Console.WriteLine(student.name["firstName"]);
	Console.WriteLine(student.name["lastName"]);
	Console.WriteLine(student.birthday);
	Console.WriteLine(student.studentID);
	Console.WriteLine(student.email[0]);
	Console.WriteLine(student.email[1]);
```

	Output:
		Hung-Wei
		Chiu
		19900317
		0156521
		sppsorrg@gmail.com
		hwchiu@cs.nctu.edu.tw

### LINQ TO JSON
在讀取方面，使用JObect.Parse來解析JSON字串，接下來在讀取資料方面，有很多種用法

- LINQ Query
- SelectToken
- dynamic Object

這邊就直接用最簡單的方法去列印JSON的資料

```c#
	string json = @"	{
	  'name': {
	    'firstName': 'Hung-Wei',
	    'lastName': 'Chiu'
	  },
	  'birthday': '19900317',
	  'studentID': '0156521',
	  'email': [
	    'sppsorrg@gmail.com',
	    'hwchiu@cs.nctu.edu.tw'
	  ]
	}";
	JObject rss = JObject.Parse(json);
	Console.WriteLine(rss["name"]["firstName"]);
	Console.WriteLine(rss["name"]["lastName"]);
	Console.WriteLine(rss["birthday"]);
	Console.WriteLine(rss["studentID"]);
	Console.WriteLine(rss["email"][0]);
	Console.WriteLine(rss["email"][1]);

```
	Output:
		Hung-Wei
		Chiu
		19900317
		0156521
		sppsorrg@gmail.com
		hwchiu@cs.nctu.edu.tw
### Modify JSON Format Data
### Serializing
這邊我沒有找到好的辦法，目前可能是要先deserialize給寫到物件，再對該物件進行操作，最後在serialize給寫回JSON去。

### LINQ TO JSON
這部分直接對JObject去進行修改就可以了

```c#
rss["studentID"]="9717164"
```

還有很多詳細的用法，包刮檔案讀取、JArray、JValue...etc
詳細的就看
[官方文件](http://james.newtonking.com/projects/json/help/#)
