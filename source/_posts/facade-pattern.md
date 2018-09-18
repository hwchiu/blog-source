---
title: Facade Pattern
keywords: 'Design Pattern, Facade Pattern'
date: '2013-11-22 15:05'
comments: true
tags:
  - DesignPattern
abbrlink: 9003
description: 本文來介紹 Facade Pateern, Facade Pattern這個模式是用來為一個已經存在的介面定義一個新的介面，這個新介面可能會擁有下列特性。1. 更高層次的抽象化 2.更容易使用. 3. 功能是本來系統的子集合。透過簡單的範例跟大家介紹 Facade Pattern 的使用情境以及如何使用

---


# Preface 
Facade Pattern這個模式是用來為一個已經存在的介面定義一個新的介面，
這個新的介面可能有下列特性
* 更高層
* 更容易使用
* 功能是本來系統的子集合



## Compare

# 使用前
![test.png](http://user-image.logdown.io/user/415/blog/415/post/161639/nVrGksQZFzRwOrUzxkgc_test.png)



# 使用後
![test.png](http://user-image.logdown.io/user/415/blog/415/post/161639/5TeQHkh1Re2r4wzaROs0_test.png)

以上圖為例子，再最原始的情況下，每個Client連線進來後都必須要跟後方三個Manager進行溝通，這樣的話對於Client方面會很複雜，同時整個系統的密合度太高。

如果透過 **Facade pattern** 設計一個介面，把與內部的溝通都隱藏起來，然後外部的client只要與這個介面溝通即可，未來若是內部有任何變動，只要針對Facade的介面去修改，Client端不必去修改也能夠正常運作。


## Examples

``` java=

#original system

public class MapManager{  
	public Point getLocation(String name){....}
 	public Point setLocation(String name, Point point) {....}
    ...
}

public class ItemManager{
	public ArrayList<Item> getItemList(String name) {....}
    public void deleteItem(String name, Item item) {....}
    public void addItem(String name, Item item) {....}
   ...
}

public class SkillManager{
	public ArrayList<Skill>	 getSkill(String name) {....}
    public void  learnSkill(String name, String skillname) {...}
   	public void  forgetSkill(String name, String skillname) {...}
    ...
}

#Facade interface

public class PersonManager{
	 private MapManger mapManager;
 	 private ItemManager itemManager;
 	 private SkillManager skillManager;
 
	 public  PersonManager (){
      mapManager = MapManager::GetSingleTon();
	  itemManager = ItemManager::GetSingleTon();
      skillManager = SkillManager::GetSingleTon();
	 }
	 public loadData(String name,Point point,ArrayList<Item> itemList,ArrayList<Skill> skillList){
			point = mapManager.getLocation(name);
      itemList =ItemManager.getItemList(name);
      skillList = skillManager.getSkill(name);

	 }
}

# Client 
 
public class Client {
	public Point point;
    public ArrayList<Item> itemList;
    public ArrayList<Skill> skillList;
    public Client(String name){
        PersonManager pm = new PersonManager();
        PersonManager.loadData(name,point,itemList,skillList);
	  }
}
```
