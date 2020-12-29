---
layout: post
title: Floodlight LLDP problem
date: '2014-05-06 05:25'
comments: true
tags:
  - SDN
  - Openflow
  - Jave
  - Floodlight
keywords: 'SDN,Controller,Floodlight'
abbrlink: 29799
---
問題來源:
[Floodlight LLDP problem](https://groups.google.com/a/openflowhub.org/forum/#!topic/floodlight-dev/15mTiLL0__A)

問題描述:

- 從floodlight去觀察，會發現有Link的`{source,dest}`都是相同的`dpid`但是不同的`port`

想法思路:

-	從該link的結果可以先猜測應該是LLDP從`port 2`送出去後不知道為什麼從`port 3`給接收到了.

- 那我想到的時候，中間兩個switch使用`傳統的learning switch`把這個`LLDP`給一路廣播下去，使得LLDP繞了回來，我覺得可能是後面兩個switch還沒有連上controller的時候會把自己運作成傳統的switch，因此我就詢問對方的網路環境.
- 在確認對方網路並非是mininet的後，就請對方先把`ovs`給設定成`secure mode`，在這種mode下，若是沒有跟controller連線，ovs就不會有任何的行為.
- 最後對方表示一切都正常了，所以發生原因應該就是中間的switch尚未變成openflow swtich前會把LLDP給透過傳統的方式給轉發下去，導致LLDP繞了回去.
