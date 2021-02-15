---
title: 鐵人賽系列文章- Day 24 - Secret 的部署問題與參考解法(下)
keywords: 'Kubernetes,DevOps,Linux,K8s'
tags:
  - ITHOME
  - DevOps
  - Kubernetes
description: ITHOME-2020 系列文章
abbrlink: 62780
date: 2021-01-31 18:00:31
---

前篇文章中我們探討了 CI/CD 自動化部屬中機密資訊的保存與使用議題，介紹了第一種透過 Pipeline 架構本身來處理問題，同時也探討了潛在的隱憂，而今天這篇我們將繼續從不同的架構下繼續探討解決方案



開始之前，我們先複習一下，我們會有三個使用情境， `Helm`, `原生 Yaml` 以及 `Kustomize`

# Centralized Management

這種架構會有一個 Secret 的管理者，這個管理者所在的位置影響不太大，主要問題是取決於什麼時間點使用這個管理者。

相較於上一篇的架構，將這個 secret 管理者從 pipeline 系統抽離的一個好處就是我們把存取機密資訊的用法與 pipeline 系統脫離，因此未來如果需要轉換到不同平台，所花費的時間與成本會少很多，因為大部分的東西都可以共用。

今天這個架構的重點是，我們將產生 secret 的發生地點往後延遲，從本來的 pipeline 系統推遲到 kubernetes 內部，當應用程式真正要部署的時候才會去動態產生這些 secret 的檔案。

上述是一個概念，但是實作上可以有不同做法，舉一個範例就是，Pod 本身先賦予相關的權限能力，讓其有能力跟 Secret 管理者溝通，然後部署 Pod 的時候要去描述說 `資料庫的密碼放在什麼地方，什麼路徑`，接下來 Pod 把這些資訊拿去問給 Secret 管理者，然後管理者回覆這些資訊。

也可以有不同做法是 Pod 本身其實不知道後面是接什麼系統，總之他相信會有人給他一些設定檔案，然後我們有額外的其他應用程式，或是 sidecar container 會幫忙去問 secret 管理者，問完後就產生出最後的設定檔案給應用程式使用。

後者的做法比前者更好，因為對於應用程式來說，我們隱藏了背後的運作細節，這使得開發者比較容易本地端開發，用一個簡單的設定檔案就可以建立環境，同時未來要遷移到不同系統時，要改變的只有維護團隊以及那些 sidecar 的運作模式，應用程式本身邏輯都不需要更動。



接下來我們嘗試將上述三種情境套入到這個架構中，看看會怎麼執行

### Helm

1. 一開始 Helm 裡面就要準備好相關的設定檔案，包含帳號密碼要用什麼東西問等，這部分取決於實作的需求
2. CI/CD pipeline 運行到後面階段後，沒有什麼事情要特別處理，因此直接透過 `helm upgrade --install ...` 的方式部署到 kubernetes內部
3. 當應用程式部署進去後，應用程式或是 sidecar 就會根據當前的設定去詢問需要的機密資訊，最後產生相對應的檔案給應用程式讀取

### Kustomize/Yaml

1. 基本上這兩者的做法一樣，其實也跟 Helm 差不多，因為不需要於 pipeline 系統中動態取得這些機密資訊，所以這個步驟也不太需要客製化。
2. 一切的運作邏輯跟 Helm 一樣，只要確保部署進去的資源能夠與 secret 管理者溝通即可。



這種架構下帶來的好處有

1. 因為與 pipeline 的部分都抽離了，所以 GItOps 就有機會實現
2. 大部分機密資訊的保存與管理都跟 pipeline 系統無關，要抽換容易
3. 這類型 secret 管理者的介面跟操作能力都比 pipeline 系統提供的介面強超級多，用起來相對彈性
4. 使用上與設計上稍嫌複雜，同時 secret 管理者的存活要自己負責，如果要設計 HA 的架構就要花一些心思去探索



![](https://i.imgur.com/3tDZr43.jpg)



# Centralized Management(ii)

這種架構是上一種的不同實作方式，就如同所說的其實 secret manager 位置放在哪邊不是重點，重點是什麼時間點去呼叫來取得資訊。

如果今天我們依然還是在 Pipeline 系統中去取得這些資訊，那基本上我們還是會得到跟第一篇架構一樣的結果。

由於這類型的服務本身也需要一些 token/key 才可以存取，而現在存取的時間點是 pipeline 系統上，所以我們只要利用 pipeline 系統本

身的管理機制去管理這些 token/key 即可。剩下應用程式所需要的機密資訊就跟遠方的 secret manager 去獲取即可。



但是這邊有一個要特別注意的地方，因為我們的 secret 管理者現在已經搬移到系統外面，所以 pipeline 系統對於你的各種操作都沒有辦

法辨認哪些東西是機密，哪些不是機密，因此 log 檔案會有機會將你的操作內容全部記錄下來，包含你的帳號與密碼。 這部分要特別小

心，避免這些資訊遺漏否則可能會釀成大禍。



之前提到的三個情境再這個架構下的實作方法大同小異，這邊就不再贅述，相關的隱憂基本上也都存在，只有搬移平台部分會是相對順

利，因為所有的存取方式與內容都與 pipeline 系統抽離。



![](https://i.imgur.com/JTgHac7.jpg)





# Encrption/Decryption

最後一個架構的做法是透過加解密的方式來記錄這些資訊，過往我們會擔心 Kubernetes Secret 的資訊是因為其本身只有 base64 編碼，並不是加密，因此實際上就是沒有安全性可言。

那如果我們有辦法把它變成加密後的結果，是否就會更有信心的放到 Yaml 上面，直接使用 Git 保存?

這個架構就是基於這樣的想法，其運作流程如下(概念，實作不一定)

1. Kubernetes 中要安裝一個 Controller，這個 Controller 會準備 key，這把 key 會用來加密跟解密
2. Kubernetes 中新增一個全新的物件，譬如叫做 Encrypted Secret，代表被加密後的 secret 資料。
3. 叢集管理者使用那把 key 將機密資料加密，得到加密後的結果，並且把這個結果寫到 EncryptedSecret 的 Yaml 中
4. 當未來這個 EncryptedSecret 被部署到叢集之中，該 Controller 就會偵測到，並且使用 key 去解密
5. 解密成功後就會把解密的內容產生一個 Secret ，這時候應用程式就可以去讀取來使用



簡單來說，上述的流程也是把取得機密內容的時間點延後到 Kubernetes 內部，只是他的做法是一開始加密，直到進入 Kubernetes 後就將其解密，最後得到真正資訊。



接下來我們嘗試將上述三種情境套入到這個架構中，看看會怎麼執行

### Helm

1. 一開始 Helm 裡面就要先透過 key 去加密，準備好一個 Encrypted Secret 的物件
2. CI/CD pipeline 運行到後面階段後，沒有什麼事情要特別處理，因此直接透過 `helm upgrade --install ...` 的方式部署到 kubernetes內部
3. 當應用程式部署進去後，該 controller 偵測到 Encrypted Secret 的出現，就會幫其解密，解密後產生對應的 secret 物件，然後應用程式去讀取

### Kustomize/Yaml

1. 基本上這兩者的做法一樣，其實也跟 Helm 差不多，因為不需要於 pipeline 系統中動態取得這些機密資訊，所以這個步驟也不太需要客製化。
2. 一切的運作邏輯跟 Helm 一樣，只要確保部署進去的資源能夠與 secret 管理者溝通即可。



這種架構帶來的好壞處有

1. 本身也不需要 CD pipeline 的參與，所以 GitOps 的概念可以實現
2. 如同上述架構一樣， Controller 的生命尤為重要，要保護好他的生命以及相關的 key
3. 機密資訊都直接存放到 Git 上面，每次要修改都要有對應權限的人去產生加密後的結果，有可能會讓工作效率比較低，但是安全性更高

![](https://i.imgur.com/Vwcbn37.jpg)



到這邊為止我們探討了數種可能的解決方式與架構，下一篇我們將實際操作一個範例來展示其用法

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

詳細可以參閱
矽谷年線上學院
https://course.hwchiu.com

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>