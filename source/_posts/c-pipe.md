---
layout: post
title: Linux下 pipe介紹
date: '2013-03-29 15:05'
comments: true
tags:
  - C
  - System
abbrlink: 52028
---

#[User]

不論是bash,tcsh,又或者是windows的cmd，都有一種叫做PIPE的功能

能夠將兩個獨立的程式給串接起來，把前面程式的輸出當作下一個程式的輸入

擁有這個指令，就能將本來當一功能的程式給組合起來變成複雜的工具了

<!--more-->


舉例來說，我想要知道我當前目錄下有多少個檔案

就可以使用ls跟wc兩個指令合作完成，

使用 ls | wc  就會將ls的結果(檔案列表)當作輸入傳給wc這隻程式，然後就可以輕鬆地算出當前目錄的檔案數量

或者是有時候想要搜尋某些特定的字串，都會使用grep這個指令，譬如想要搜尋某個特定使用者正在執行的所有程序

ps auxww | grep username

所以pipe對於系統管理來說，是個非常重要的概念，能夠將每個獨立細小的程式給串接起來完成複雜的工作。

#[程式設計]

在FreeBSD(linux)上，shell能夠辦得到這樣的功能，實際上是利用了kernel中pipe的功能，這邊就已linux kernel 3.5.4為架構。

在程式中，pipe的概念就是一個水管，這個水管有兩個端口，一端負責寫資料到pipe，一端負責將資料從pipe中讀出來，所以我們可以做個簡單的測試。

	int main(){
		int rand1,rand2;
		int fd[2];// declare a two-d array, store file_descriptor of the pipe (two side)
				  // fd[0] mease read side, fd[1] means write side
		pid_t pid;//child process的pid
		pipe(fd); //call system call (pipe) to create a pipe
		//use fork to create a child process
		//child process will wrtie data to pipe, and parent will read data from pipe
		//child process
		if((pid=fork())==0){
			srand(getpid());
			close(fd[READ_END]);//child won't use read size, so close it
			rand1=rand()%RANGE; //create random number
			write(fd[WRITE_END],&rand1,sizeof(rand1)); //write to pipe
			close(fd[WRITE_END]);//close write side when write over
			printf("%d has been created In Child Process \n",rand1);
			exit(1);
		}
		else if(pid>0){
			srand(getpid());
			close(fd[WRITE_END]);//parent won't use write size, so close it。
			rand2=rand()%RANGE;//create random number
			read(fd[READ_END],&rand1,sizeof(rand1));//read the data from pipe
			printf("%d has been created In Parent Process \n",rand2);
			wait();
			printf("Parent Process calulate sum is :%d \n",rand1+rand2);
			close(fd[READ_END]);//close read side
			exit(1);
		}
	return 0;
	}


>執行結果:  
>8 has been created In Child Process

>5 has been created In Parent Process

>Parent Process calulate sum is :13

>----------------------------------------

>3 has been created In Child Process

>3 has been created In Parent Process

>Parent Process calulate sum is :6

 

實際上，如果想要對同個端口去進行寫跟讀的動作，是行不通的，乍看之下會覺得PIPE只是一個

buffer，放置資料而已，實際上在kernel中，pipe被視為是一個file，當我們呼叫pipe時，真正最後會

呼叫到do_pipe這個function，在這個function中，會針對pipe的兩個端口分別去設定

O_RDONLY;O_WRONLY的標籤，這樣的設定使得pipe的端口就真的一邊只能讀，一邊只能寫。

有空在來講述一下file_descriptor file file_operation三者的關係，以及到底 file,socket,pipe...等這些device到底在kernel中如何運作。
 
