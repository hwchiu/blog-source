---
layout: post
title: '淺談auto_ptr'
date: 2013-03-29 14:18
comments: true
tags:
   - C
---


在寫C++的時候，常常會使用new來獲取heap的空間，來取得heap的空間，如下。
``` c++
	void Test1(){
	   char* name = new char(100);
	   //process something
	   delete name;
	}
	char* GetHeap(char* name){
	   char* name = new char(100);
	   return name;
	}
```
然而對程式設計師來說，最麻煩的過於一旦new出空間後，一定要執行delete把空間回收，以免發生memory leak的行為
 
<!--more-->


>memory leak: 某塊記憶體再也沒有檔案去reference,當妳new出空間後，沒有去delete回收空間時，很容易發生

	void Test2(){
	char* name = new char(100); //要空間
	}//沒有回收，就會造成剛剛取得的空間會變成記憶體中的孤兒

所以在學習 new/delete時，總是被不斷的叮嚀，有new就要有delete，一定要成對出現。

在C++的標準中，提供了一個叫做auto_ptr的標準，就是專門用來處理這種情況，

只要當這個指標沒有繼續reference時，便會自動回收自己，讓程式設計師更方便，能夠遠離new/delete之苦。

可惜如此良好想法的設計，在目前C++標準中，卻是不建議被使用的，因為其某些特性，反而會使得程式碼變得難以捉模。

***
以下就來正式介紹 **auto_ptr**

俗稱智慧型指標，保證只要本身被摧毀，必定釋放其所指資源

是一種指向所屬物件擁有者的指標。所以當身為物件擁有者的auto_ptr被摧毀時，該物件也會被摧毀

auto_ptr要求，一個物件只能有一個擁有者，嚴禁一物二主(**這點卻造成困擾**)

本身不支援指標算數(++之類)

不允許一般指標慣用的賦值初始化方式，必須直接使用顯示轉換來初始化，因為在其實作中，有使用到explicit關鍵字。

``` c++
void Test3(){
   auto_ptr<int> ptr(new int[100]);  //ok
   auto_ptr<int> ptr = new int[100];  //error
   //process something
}
```
 
#擁有權的轉移
auto_ptr 所界定的是嚴格的擁有權概念，由於一個auto_ptr會刪除所擁有的物件，不應該發生同時間有多個auto_ptr共同擁有一個物件

否則就會出現問題，程式設計師要特別小心避免寫出這樣的程式碼，
``` c++
	auto_ptr<int> ptr1(new int[100]);  //declare a auto_ptr pointer toint
	 *ptr1=123;                        //change value
	 cout<<*ptr1<<endl;
	 auto_ptr<int> ptr2(ptr1);         //initial ptr2 via ptr1 and ptr1 trans its ownership
	 if(ptr1.get()==NULL)
	    cout<<"ptr1 has transfered ownership to ptr2"<<endl;
	 cout<<*ptr2<<endl;
	 return 0;

	------------------
	output:
	123
	ptr1 has transfered ownership to ptr2
	123
```


執行第二行的時候，ptr1會把所有權轉移給ptr2,所以此行一旦結束，ptr1就會是個null。

同樣的問題也會發生在assign的情況下
 
``` c++
	auto_ptr<int> ptr1(new int[100]);
	auto_ptr<int> ptr2;
	ptr2 = ptr1 ; //transfers ownership from ptr1 to ptr2
```


此外，如果ptr2被賦值前擁有另外一個物件，則被賦值後，便會釋放該物件，並呼叫destructor。

``` c++
	#include<iostream>
	#include<memory>
	using namespace std;
	class Student{
	    public:
	      Student(int index):_index(index){
	       cout<<"Student"<<_index<<" constructor"<<endl;
	      };
	      ~Student(){
	       cout<<"Student"<<_index<<" destructor"<<endl;
	      }
	    private:
	      int _index;
	};
	
	int main()
	{
	 auto_ptr<Student> ptr1(new Student(1));
	 auto_ptr<Student> ptr2(new Student(2));
	
	 ptr2 = ptr1 ;  //ptr2's object will release
	 cout<<"over"<<endl;
	 return 0;
	}

	------------------
	output:
	Student1 constructor
	Student2 constructor
	Student2 destructor
	over
	Student1 destructor
```

#缺陷

由於auto_ptr本身就涵蓋擁有權，如果無意去轉換擁有權，絕對不要在參數中使用auto_ptr，也不要另auto_ptr作為返回值，否則會有很大的災難。以下是個範例
``` c++
	#include<iostream>
	using namespace std;
	void bad_print(auto_ptr<int> p)
	{
	   if(p.get()==NULL){
	     cout<<"NULL"<<endl;
	   }
	   else
	     cout<<*p<<endl;
	   }
	int main()
	{
	   auto_ptr<int> ptr(new int);
	   *ptr=123;
	   bad_print(ptr);
	   *ptr=456;
	   return 0;
	}

	-------------
	output:
	123
	Segmentation fault (core dumped)
```
因為在參數中，使用了auto_ptr，所以當呼叫此function的時候，便會把所有權轉移到function中的臨時變數，然後當function結束後，

這個區域的臨時變數又會銷毀，而在main中的ptr,因為呼叫function後擁有權轉移，所以第二次執行賦值的動作，就會出現runtime error了，  

因為此時ptr並沒有任何指向任何物件，所以導致此崩壞行為。

在這種情況下，我們可以考慮採用pass by reference的方式來傳遞變數，可惜對於auto_ptr來說反而會製造更多麻煩，更難去掌握擁有權轉移的順序，

因此能避免就盡量避免使用auto_ptr跟pass by reference。

如果今天真的有要當參數傳入的需求，這時候可以使用**constant reference**,如此一來可以使得auot_ptrs本身無法轉移所有權。

以下的例子就會編譯錯誤，可提醒設計師轉移權的問題。與一般不同的是，這邊的const代表的並不是不能修改指標所擁有的物件，而是不能更改

auto_ptr的擁有權，更像是T* const ptr;
``` c++
	#include<iostream>
	
	using namespace std;
	void bad_print(const auto_ptr<int> p)
	{
	   if(p.get()==NULL){
	    cout<<"NULL"<<endl;
	   }
	   else
	    cout<<*p<<endl;
	}
	int main()
	{
	   const auto_ptr<int> p(new int); 
	   *p = 123;  
	   bad_print(p); //COMPILE TIME ERROR
	   *p = 456;
	   return 0;
	}
```
總結
auto_ptrs之間不能共享擁有權

auto_ptr間沒辦法同時擁有一個物件，因此當把兩個auto_ptr指向對方時，就會使得本來的一方釋放其所擁有的物件，之後若是再透過該指標去操作，就會出現錯誤。

並不存在針對array設計的auto_ptr。

auto_ptr的內部設計是delete,而非delete[],所以不可以指向array。 
auto_ptr並非萬能指標。

由於auto_ptr並非一個計數型指標(或者是上限為一的計數型指標)，因此在使用上有非常多的限制，如果設計師沒有完全瞭解其特性，很容易就會讓程式出錯。 

千萬別在STL 容器中使用auto_ptr。

因為STL標準規定，C++標準容器容易必須要符合"copy-constructible" 跟 "assignable." ，亦即容器中的元素必須都要能夠被複製跟賦值，然而auto_ptr的特性並不相容上述行為，所以切忌使用，否則容易出錯。

