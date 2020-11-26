---
title: '[Golang] aggregate in mongo'
keywords: 'golang,mongo,aggregate,mgo'
abbrlink: 58018
date: 2018-08-31 14:09:05
tags:
  - Golang
  - Mongo
description: 本篇文章首先跟大家分享一個常見的 Schema 設計，在此情境下，為了讀取一連串的資料，我們有不同種方式可以辦到。其中一種就是本文的主角, Aggregate 的概念。為了解釋 Aggregate 如何運作以及如何實現，本文採用 Golang 作為基本的程式語言，並且使用 mgo 作為與 mongo 進行處理的第三方函式庫。此外也採用了最原始的讀取方式，並且將此方式從撰寫方式以及效能兩方面直接與 Aggregate 進行比較。

---

在探討整個主題之前，我們先設計一個簡單的 `schema` 來符合這次
假設今天在資料庫內有主要結構，分別是 `User` 跟 `Pod`
`User` 非常簡單，就是描述一個使用者
`Pod` 這邊不用在意他是什麼東西，他是一個資源，然後透過 `User` 創立的，所以每個 `Pod` 裡面都會有一個欄位去記錄是哪個 `User` 創立的。

有了前述的假設之後，接下來想像一個實際需求
1. 有一個 `UserA` 創造了數百個 `Pod` 資源。
2. 想要 Query 該 `UserA` 並且列出該`User`創造的所有`Pod`資源的細部資料

![Imgur](https://i.imgur.com/rFmSBdw.png)


在以前 SQL 的時候，我們可能會透過 `SELECT` 搭配 `JOIN` 的方式來存取，那今天假如在 `mongoDB` 的環境下，我們可以怎麼處理?

本文章的所有範例程式碼都可以在[mgo Aggregate Example](https://github.com/hwchiu/GolangPratice/blob/master/mgoAggregate/main.go)找到

必須要注意， mongodb 的版本至少要 > 3.0 之後才支援本文描述的 `Aggregate`


## Structure
這邊就採用 `golang` 簡單的描述一下我們的結構，這些結構會用在 `mongodb` 裡面
```go=
type User struct {
	ID   bson.ObjectId `bson:"_id"`
	Name string        `bson:"name"`
	Pods []Pod         `bson:"pods"`
}

type Pod struct {
	ID        bson.ObjectId `bson:"_id"`
	CreatedBy bson.ObjectId `bson:"createdBy"`
	Name      string        `bson:"name"`
}
```

## Find
第一種方法其實非常直覺
1. 就是先取得該 `User` 的實例
2. 去 `Pod` 的 `Collection` 內進行搜尋，找出所有的 `Pod` 其記錄的`User`是` UserA`.

假設我們已經有該 `User` 的物件了，接下來很簡單透過 `find` 的方式找出所有由該 `User` 所創造的 `Pod`.
並且把找到的結果都放到 `User`  物件裡面的 `Pods` 變數
```go=
c := db.C("pods")
c.Find(bson.M{"createdBy": user.ID}).All(&user.Pods)
```

## Aggregate
在 `mgo`裡面，如果想要做到 `Aaggregate` 類似這種 `Join` 的方式，其實可以透過一個叫做 `pipeline` 的函式來完成

為了使用 `pipeline` 的，我們必須要先描述我們想要如何去尋找這些資料，這些資料是由`bson.M{}` 陣列組成的。
該陣列內，至少要提供兩種不同的查詢方式
1. $match: 這邊用來描述我們要如何查詢 `User` 物件，在此範例中我們使用特定的`User.ID`來描述該 `User`
2. $lookup: 用來描述要如何 `Aggregate`, 其中有四個欄位要填寫
    - from: 用來進行 `Aggregate` 的 `collection`  名稱，此範例中我們設定為 `pods` 這個 `collection`.
    - localField: 在該 `pipeline` 要使用的 `collection` 內要使用的欄位，這邊我們會使用 `User` 裡面的 `_id`.
    - foreignField: 在 `from` 這個 `collection` 裡面要使用的欄位，此範例就是 `Pods` 裡面的 `createdBy`.
    - as: 這個是用來存放查詢後的資料，首先其本身必須是個`Array`，同時該欄位必須定義在 `User` 結構裡面。



程式碼如下，該程式碼的意思很簡單，我們對 `users` 這個 `collection` 裡面透過 `$match` 去尋找特定的 `User`，同時透過 `$lookup` 的方式到 `pods` 該 `collection` 裡面去把 `User.ID == Pods.CreatedBy` 相同的 `pods` 都找出來，最後放到 `user.pods` 裡面
```go=
c := db.C("users")
pipeline := []bson.M{
    {"$lookup": bson.M{"from": "pods", "localField": "_id", "foreignField": "createdBy", "as": "pods"}},
    {"$match": bson.M{"_id": user.ID}},
}

var resp User
err := c.Pipe(pipeline).One(&resp)
if err != nil {
    fmt.Println(err)
}
```

## Performance
因為這次的範例非常簡單，所以並不能用來代表在所有的使用情境下的結果，只能用來表示如本文的情境下的效率問題。

在上述的範例專案內，一開始會先隨機產生一個使用者，並且隨機產生`30000` 筆 `Pod` 資料，並且嘗試比較不同數量的情況下，透過 `Find` 以及 `Aggregate` 實際上的效能問題。

下列的時間都是 **(ms)**

| Methond\Numbers | 1000 | 5000 | 10000 | 30000 | 50000 |
| - | - | - | - | - | - |
| Find | 5.7 | 18 | 35 | 111 | 250 |
| Aggregate | 7.4| 22| 77 | 180 | NaN |


這邊要特別注意的是，當 `Pods` 的數量過高的時候，使用 `pipeline` 來處理就會得到下列的錯誤訊息

`Total size of documents in pods matching { $match: { $and: [ { createdBy: { $eq: ObjectId('5b86dac74807c532d70bea52') } }, {} ] } } exceeds maximum document size
`

根據官網描述，預設的情況下，只能存放 `16MB` 的資料。
這意味者資料過多的情況下，使用者要注意這個現象，避免資料存取失敗。


## Summary
本文假設的情境非常簡單，基本上兩種方法都可以完成，但是在資料多寡的情況下，花費的時間就有所區別。
若是有更複雜的需求，在 `pipeline` 裡面還可以設定除了 `$lookup` 以及 `$match` 以外的用法，透過一次的呼叫就把資料給查詢完畢。
至少如果只是本文這種很簡單的情境，其實自己額外查詢即可，不論在簡潔性跟效率上我認為都更高。

所以還是一樣，不同情境下，每個功能都會有不同的使用方法跟考量點。
這邊就讓各位自己去評估囉


## Reference
- https://docs.mongodb.com/manual/aggregation/

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4N7G

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