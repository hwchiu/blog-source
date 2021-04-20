---
title: N-Queen problem
keywords: 'NQueen,N Queen, Algorithm, c++'
date: '2014-09-19 09:01'
comments: true
tags:
  - Algorithm
abbrlink: 42732
description: 本文介紹一個常見也是著名的演算法題目， N-Queue(N皇后),該題目假設在一個 NxN的棋盤上面，想求得總共有多少種的辦法去放置 N 的皇后且這些皇后彼此都不會互相影響，影響的定義則是上下左右直線，以及左下右上斜線總共六條斜線中都不能有其他的皇后。 這個經典問題有非常多種的解決方法，最簡單的就是透過遞迴配上剪枝來加速，然而在N過大的時候，盤面的數量太多，每次的運算都太花時間會導致整體求解速度過慢。因此本文使用基於 Bit 為單位的方式去進行運算，能夠更快速的求出答案。

---

# Preface
在成功嶺新訓得十六天，為了不讓自己的腦袋沒有運轉，上課的時候我就到處找東西想，腦中第一個想到可以動腦的題目就是Ｎ皇后


N皇后是一個經典的題目，最初是8皇后，後來被推廣至Ｎ皇后，想要瞭解此題目的規則可以參考[八皇后問題](http://zh.wikipedia.org/wiki/%E5%85%AB%E7%9A%87%E5%90%8E%E9%97%AE%E9%A2%98)

此題目最直覺的解決方法就是Backtracking，去枚舉所有可能性，對於每種可能性檢查是否合法，這部分可以透過一些機制來達到剪枝的效果，藉此減少運算次數已提升運算速度。
由於之前跟Ensky一起合作寫[大地遊戲產生器](https://github.com/ensky/GroundGame-scheduler)時，我有嘗試使用bitwise的架構來進行加速，因此這次我決定在成功嶺上課的時間，想辦法把N皇后的問題也用bitwise的方式來處理。

雖然以前就知道N皇后有用bitwise的方式來運算，但是這次想要靠自己慢慢思考，去想出每一個步驟來解決此問題。因此我就在成功嶺上課的過程中，將程式碼給手寫下來，待新訓結束後趕緊來測試結果。

# Algorithm

在原本沒有加速的版本中，我們會使用**2D Array**來表示當前的盤面，每次選擇下一個皇后位置時，都會根據此盤面來選擇一個合法的位置。再選擇一個合法的位置時，不但要確認該col上是否已經有放過，還要檢查是否有斜線通過，這部分算法中比較麻煩的部分。
再**Bitwise**的版本中，我們不使用**2D Array**來表示盤面，而是使用一個整數的bit來表示當前盤面的狀況，利用0與1分別代表是否有放置皇后，舉例來說，以6X6的盤面來講，**001001**就代表者第三個col跟第六個col上都有皇后存在。這邊我使用了三個變數，分別是**position**,**left**,**right**三個變數來記錄盤面上的可能性。**position**紀錄的是皇后在哪些col上面有放置，**left**紀錄的是所有皇后左對角線的位置而**right**則是所有皇后右對角線的位置。
以下面5X5的圖為例。
```
10000
00100
00001
01000
00010
```
此圖中1代表皇后的位置，所以第一個row中，皇后放到第一個col，第二個row中，皇后放到第三個col的位置，以此類推。在這種情況下，我們要如何用剛剛所說的三個整數來描述此情況。這邊我們會採取漸進式的方式來描述此三個變數，一開始只有第一個row，接下來是包含了前兩個row，再來就前三個row，以此類推。

## 第一回合
Position: 10000
Left: 00000  (因為1的左對角線就超出邊界了，所以此時的Left就是空的）
Right: 01000
## 第二回合
Position: 10100 (因為第二個row的皇后是放在第三個col，因此就跟上一回合的Position給結合)
Left: 01000
Right: 00110 (之前的Right到了這一個回合，要先往右邊移動一格，變成00100，然後第二回合的皇后所產生的右對角線則是00010，將這兩個集合起來就會變成00110)
## 第三回合
Position: 10101
Left: 10010 (上一回合的Left到此回合後，就要往左移動一格）
Right: 00011
## 第四回合
Position: 11101
Left: 10100
Right: 00101
...以此類推
所以目前已經想好了要如何使用三個整數的bit來表示當前盤面的情況，如果要挑選一個合法的位置，首先將**Position**|**Left**|**Right**給得到一個新的值，這個值中為0的bit就代表是可以用的地方，因此從這些地方來挑選下一個合法位置即可。
以剛剛的範例來說，過的第一回合後，
Position: 10000
Left: 00000
Right: 01000
將三個變數取ＯＲ得到新的值（101000），此值代表了第二回合要選取皇后時，第一跟第三個col都不能放皇后，只能放2,4,5,6四個col。
這些表達方式都釐清後，接下來最困難的就是如何自動的選出合法位置，以剛剛的範例來說(101000)，針對此範例，我們知道皇后只能放在2,4,5,6四個col，我希望能夠找出一種方法來依序取出這四個位置，並且在四個位置都去放下皇后試試看。
這邊最直覺的方法就是將該值(101000)不停地**/2**與**%2**，如此一來就可以知道每個bit是0還是1，但是這種方法的卻點就是要跑太多次迭代了，我希望能夠找出一個方法，該數值中有多少個0，就迭代多少次，不需要額外的次數來處理。
原本數值:**101000**
## 第一次迭代
這次選出了第一個0，（000001)
因此當前的Position就會變成 (101000) | (000001)  = 101001
## 第二次迭代
這次選出了第二個0，（000010)
因此當前的Position就會變成 (101000) | (000010)  = 101010
## 第三次迭代
這次選出了第三個0，（000100)
因此當前的Position就會變成 (101000) | (000100)  = 101100
## 第四次迭代
這次選出了第四個0，（010000)
因此當前的Position就會變成 (101000) | (010000)  = 111000
除了更新Position，Left以及Right也都要一併更新。簡單來說，我希望依序取出bit中是0的位置。
這邊我使用了二補數的概念來處理，如下。
由於使用的是一個32bit的整數來處理，但是實際上對於8X8盤面只需要8個bit，因此在計算時，必須要注意沒有使用到超過8bit以後得值，這邊我用一個limit (11111111)來作為一個極限的判斷。
``` cpp
	currentPos = position | left | right;
	int newPos;
while( currentPos < limit){
  newPos = (currentPos+1) & ~currentPos;
  newPosition = position | newPos;
  left = limit & ((left | newPos) <<1);
  right = limit & ((right | newPos) >>1);
  //dosomething
  currentPos = currentPos | newPos;

}
```
這邊會將每次選到的位置都放到**newPos**，並用此變數來更新**Position**等變數，
最後使用**currentPos**來記錄還有哪些位置可以選，如此一來就能夠依序的把bit為0的位置取出來。

整個程式的完整程式碼如下，執行時輸入N，此Ｎ代表的是該局為NXN的皇后。

``` c
#include<iostream>
using namespace std;
int limit;
int N;
int counter;
void DFS(int position,int left, int right,int depth){
    if(depth == N){
        counter++;
        return;
    }
    int currentPos = position | left | right;
    int newPos;
    while( currentPos < limit ){
        newPos = (currentPos+1) & ~currentPos;
        DFS( position | newPos, limit & ((left | newPos)<<1), limit&((right | newPos)>>1),depth+1);
        currentPos = currentPos | newPos;
    }
}

int main(){
    cin >> N;
    counter =0;
    limit = (1<<(N))-1;
    DFS(0,0,0,0);
    cout << counter<<endl;
}
```

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

線上課程詳細資訊: https://course.hwchiu.com/
另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>
