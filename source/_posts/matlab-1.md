---
layout: post
title: Matlab 簡單練習
date: '2013-03-29 15:04'
comments: true
tags:
  - Matlab
abbrlink: 25152
---


應朋友的要求，用matlab幫忙寫了一個簡單的腳本

需要能夠ˋ彈出對話框選擇一個資料夾，讀取資料夾底下的影像檔，然後與某個特定的影像檔做相減，並命名輸出

這部分用到了一些指令，在這邊紀錄下來


``` matlab
%choose directory
target_path = uigetdir();
file_path = [target_path '/C00*.tif'];
background = [target_path '/REF.tif'];
file_struct = dir(file_path);
back_struct = dir(background);

%load background image
back = imread([target_path '/' back_struct.name]);
for i=1:length(file_struct)
temp_image = imread([target_path '/' file_struct(i).name]);
result_image = imsubtract(temp_image,back);
imwrite(result_image,[target_path '/new' file_struct(i).name]);
end
```
====END=======

<!--more-->



首先使用到了uigetdir,與其類似的還有uigetfile

呼叫此函數後，會彈出directoryOpenDialog的介面，選擇完畢後，會把選擇的路徑回傳

接下來我想要移動到該路徑，於是希望透過 cd 這個指令，無奈 cd這個指令沒有辦法吃參數，只能吃完整路徑，所以就必須要改換成其他的方法

由於我已經知道圖檔的命名規則，於是先用 [] 的方式，把字串給連接起來，這邊使用regular的方式，之後再搜尋檔案的時候會更方便

接者使用dir這個指令，就可以得到我想要的所有檔案，dir回傳的是一個struct，內容包含了檔案的

>name    -- 檔案名稱
>date    -- 修改日期
>bytes   -- 檔案大小
>isdir   -- 是否為資料夾
>datenum -- Matlab特定的修改日期

這邊我只需要它的名稱，於是透過一個迴圈，把所有的路徑檔案都以圖片的方式(imread)給讀取近來
在與事先讀取好的背景圖片(back)使用imsubtract做相減，得到新的圖片，再透過imwrite給寫出檔案
