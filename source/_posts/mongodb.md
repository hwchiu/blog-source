---
layout: post
title: MongoDB
date: '2014-01-06 08:32'
comments: true
tags:
  - System
  - Database
  - Python
abbrlink: 51937
---
最近在使用 **MongoDb**這套 nosql的DB,簡單紀錄一下過程

### Install
Refet to [MongoDB Installation](http://docs.mongodb.org/manual/installation/)

<!--more-->

### Manipulate
在 **MongoDB**中

![mongodb.png](http://user-image.logdown.io/user/415/blog/415/post/174500/nOcR7hCKTCyrysDk3xgh_mongodb.png)

- db: 就是database
- collection: 就是以前看到的Table
- document: 就是以前看到的record

在操作 **collection**的時候，不需要事先定義每個 **document**的欄位以及屬性，每個 **documents**的欄位數量可以不同，並且可以共存於同一張 **collection**之中。


#### Mongob Command Line

**DB**
- show dbs : 顯示目前有哪些db
- db: 顯示目前使用哪個db
- use xxx: 切換到哪個db (若不存在，就會新增)

**Collection**
- show collections : 顯示當前db下有哪些 **collections**
- db.collection.command:  **collection**的操作都是按照 `db.${collection_name},${command}`
	* db.test.insert( { "key1":"value1", "key2":"value2"}) : 增加一個新的 **document**,如果沒有該 **collections**,就會順便產生新的
  * db.test.drop() : 刪除該 collection
  * db.test.remove( { "key1":"value1", "key2":"value2"}): 刪除特定的 **document**

More refer to [MongoDB Shell](http://docs.mongodb.org/manual/reference/method/)

#### Use python

目前是用 **PyMongo**這個第三方套件

**Install**
- pip install pymongo
- Refer to [Pymongo Installation](http://api.mongodb.org/python/current/tutorial.html)

**Connection**
- client = MongoClient('localhost', 27017) : 與mongodb server連線

**Manipulate**
- Refer to [Pymongo Tutorial](http://api.mongodb.org/python/current/tutorial.html)
