---
title: "Automatically Renew Your Certificated in kubernetes by Cert-Manager"
keywords: 'kubernetes,tls,ssl,certificated,lets encrypt,cert-manager'
tags:
  - Kubernetes
  - Security
abbrlink: 772
date: 2018-10-03 02:48:04
description: 在這個資訊安全意識稍微抬頭的世代，網站配有 HTTPS 可說是個基本標配。同時因為 Let's Encrypt 的出現，讓 TLS 憑證的申請變得簡單且容易上手。然而使用 Let's Encrypt 本身還是有一些限制要處理，譬如需要定期更新憑證，以及如何驗證申請者目標網域的擁有者，這部分的操作都有對應的腳本來處理。然而在 Kubernetes 叢集之中，除了手動去運行這些腳本之外，有沒有更方便的方式可以整合這一切。本文要介紹一個叫做 `Cert-Manager` 的解決方案，透過其原理的理解，以及實際操作的步驟來學習如何更方便的在 kubernetes 叢集內管理憑證

---

# Preface
之前於 [11個保護你 Kubernetes 集群的技巧與觀念(上)](https://www.hwchiu.com/k8s-security-11tips-i.html) 中有不斷的強調加密連線 `SSL` 的重要性。最簡單的範例就是如果今天有一個需求需要於 `kubernetes` 叢集中部屬一個網頁服務供外部存取時，預設情況下我們都會走 `HTTP(80)`, 然而在安全意識崛起的時代，基本上一個網頁服務沒有 `HTTPS(443)` 的存取方式時，大家對於是否要在該網站輸入一些機密資料都會變得小心翼翼甚至拒絕使用。

為了讓該網頁伺服器提供 `HTTPS` 的連線方式，我們必須要提供一組憑證，且該憑證的簽署方是要被信任的。
在這個情況下，我們可以像第三方廠商購買憑證的方式來取得一個合法的憑證，並且將該憑證部屬在我們的網頁伺服器上，這樣就可以順利的提供 `HTTPS` 安全連線方式。

但是更多時候，很多人都只有買域名卻沒有買憑證，解決方法都是透過免費的 `Let's Encrypt` 去簽署憑證，然而 `Let's Encrypt` 本身期限很短，需要定期更新來確保憑證的有效性。

這個過程其實不會太難，基本上就是申請 `Let's Encrypt` 服務，然後想辦法定期更新該憑證即可。
然而這個步驟有沒有辦法變得很簡單? 可以讓叢集部屬人員可以用非常簡單的方式來達成這件事情?

本文會跟大家介紹 `Cert-Manager` 這套用來管理憑證相關的服務，如何透過這個服務可以輕鬆的在 `kubernetes` 內把上述的事情自動處理完畢，讓你只需要專心處理自己的應用程式即可。

# Component
如同前言所述，接下來的文章都會針對 `Cert-Manager` 這個套件作介紹，大致上會講述如果要完成前述所需要的自動更新憑證等相關事情，於 `Cert-Manager` 中實際上會是怎麼完成的。同時身為系統管理員，需要準備哪些相關的資訊來處理。

## Issuers
`Cert-Manager` 裡面總共有兩種類型的 `Issuers`, 分別是 `Issuer` 以及 `ClusterIssuers`. 這兩個的差異只有在於其影響的 `kubernetes namespace` 的範圍而以，其餘的功能都一樣. 所以接下來的文章將會使用 `Issuers` 來當做範例介紹.

`Issuers` 代表的是一個能夠從會簽署 `x509` 憑證的授權單位所取得憑證的一個資源，最
目前 `Issuers` 支援的 `DNS Challenge` 有兩種，分別是 `HTTP-01` 以及 `DNS-01`.
常見的後端代表就是想辦法從 `Let's Encrypt` 取得憑證.
1. Issuers 是透過 `Kubernetes CustomResourceDefinition(CRD)` 所自定義的新資源
    - 這意味你可以透過 `kubectl get issuers` 來檢視相關的設定
3. 對於每個 `kubernetes` 叢集來說，系統內至少要部屬一個 `Issuers` 來完成憑證的功能
4. Issuers 目前提供了四種後端接口, 這邊會稍微介紹一下彼此的差異，最後說明本文會基於哪種後端接口來展示

### ACME
這個類型應該是最常見也是最多人在使用的，透過 [Automated Certificate Management Environment(ACME)](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment)
這個協定與已知的 `CA` 溝通來取得憑證。

與 `CA` 的溝通過程中，最困難也是最核心的部份就是 `DNS Challenge`, 使用者必須要向 `CA` 證明自己擁有想要簽署憑證的域名。
目前 `Issuers` 支援的 `DNS Challenge` 有兩種，分別是 `HTTP-01` 以及 `DNS-01`.

#### HTTP-01
顧名思義，就是透過 `HTTP` 的方式來驗證是否擁有該網域，要採用這種方式基本上要滿足幾個條件
1. 首先針對想要簽署的網域進行設定，譬如將 `test.hwchiu.com` 指向到一個 `IP` 地址
2. 該 `CA` 會嘗試透過 `HTTP` 的方式去連線 `test.hwchiu.com`. 其會預期得到一個對應的 `HTTP` 回應，該回應的格式可以參考 [ HTTP Challenge](https://tools.ietf.org/html/draft-ietf-acme-acme-07#section-8.3)

透過上述兩個條件，我們可以觀察到幾個現象
1. `HTTP-01` 並不支援 `wildcard` 的 `domain`, 因為要透過 `DNS` 查詢對應的 IP 來取得 HTTP 回應
2. 為了讓 `HTTP` 封包可以順利的連線，通常客戶端都會需要有一個可存取的對外 IP 地址
    - 當然這部分你若熟悉網路，要透過 `DNAT` 的方式將封包導入到私有 IP 的網頁伺服器上也是沒有問題

大部分人的人在使用 `Let's Encrypt` 時基本上都是透過 `HTTP-01` 這個方式來驗證，因為使用上其實相對簡單，基本的那些網頁伺服器以及對應的 HTTP 格式都有相關的軟體幫忙處理，大幅度簡化了管理員的設定。

本篇文章並不打算使用這個方式，因為過於簡單沒有什麼好設定的，所以我們來看看另外一種挑戰， `DNS-01`.

#### DNS-01
相對於 `HTTP-01` 透過 `HTTP` 連線來驗證是否有該網域的擁有權， `DNS-01`  則是透過創造一個預期內容的 `TXT` 紀錄來證明你擁有該網域的擁有權。為了達成上述的事項，`Issuers` 目前有支援下列的 `DNS` 供應商來幫忙完成這個自動創建 `TXT` 紀錄的需求
- Google CloudDNS
- Amazon Route53
- Cloudflare
- Akamai FastDNS
- RFC2136
- ACME-DNS

透過 `DNS-01` 的機制就可以打破 `HTTP-01` 的限制，客戶端本身即使位於私有網路內也是沒有問題的。

而因為我自己的網域目前是放在 `Cloudflare` 上，因此本文就會採取 `DNS-01` 的方式並且使用 `Cloudflare` 作為測試的 `DNS` 提供者。

### CA
相對於上述的 `ACME` 的使用方式， `CA` 類型的 `Issuers` 則是會使用一組事先準備好的簽署金鑰來發行憑證。這組簽署憑證必須要以 `kubernetes secret` 的形式存於 `kubernetes cluster` 中。
舉例來說，你可以先透過 `openssl` 的方式產生對應的檔案，並且把該檔案加入到 `kubernetes secret` 內，最後透過 `CA` 類型的 `Issuers` 來使用。


### Valut
如果你系統內有安裝 [HashCrop Valut](https://www.vaultproject.io/) 的話，可以透過這個類型的 `Issuers` 來與 `Valut` 溝通。

### Self-Signed
這個選項目前會用到的機會不多，大致上有兩種情境
1. 想要透過 `kubernetes` 打造一個 `Public Key Infrastructur (PKI)` 架構
2. 搭配後續會介紹的 `Certificate` 作為一個 `root CA` 供 `CA` 型態的 `Issuers` 使用
    -. 可以參考 [resource-validation-webhook](/https://github.com/jetstack/cert-manager/blob/834fda15a1160b3e75d92ab7d84b1402eac9be0a/docs/admin/resource-validation-webhook.rst) 的使用範例來瞭解如何搭配使用


想要更加深入的瞭解這些選項可以直接到[官網](http://docs.cert-manager.io/en/latest/reference/issuers.html)閱讀相關的資訊以及範例使用


## Certificate
看完了 `Issuers` 之後，我們來看看第二個也是最後一個透過 `kubernetes CRD` 所建立的物件，`Certificate`.

`Certificate` 就如同其名稱一樣，代表的是一個憑證，其會嘗試使用 `Issuers` 來獲得對應的憑證與金鑰，當成功取得對應的資源後，就會將該對憑證與金鑰的資訊放到 `kubernetes secret` 裡面。
`Certificate`  以及產生的 `kubernetes secret` 都跟 `namespace` 有綁定關係，所以對於不同的 `namespace` 的應用程式若想要擁有這張簽署的憑證，則都需要創立一個對應的 `certificate` 來使用。

## Summary
基本上整個 `cert-manager` 的關係都是由 `Issuers/Certificate` 所組成的，當然也有部屬安裝 `cert-manager` 時會佈署的 `kubernetes deployment`, 該 `deployment` 作為 `Issuers/Certificate` 之間的橋樑， 定期確認 `Certificate` 的合法性並且當憑證快要到期時還會嘗試更新。

透過這些資源的整合，`Cert-Manager` 就能夠於 `kubernetes` 叢集內提供可以從 `Let's Encrypt` 取得憑證並且定期自動更新以確保憑證不會過期的憑證供我們其他的應用程式使用。

# Setup
看了上述的基本概念介紹後，我們接下來就要手把手的嘗試透過 `Cert-Manager` 來取得一個憑證，這中間的過程大概如下
1. 部屬 `cert-manager` 相關的服務到 `kubernetes` 叢集中
2. 部署 `Issuers`
3. 部屬 `Certificate`
4. 透過檢查 `Kubernetes Secret`, 確認是否有拿到可用的憑證

整個系統的架構圖如下，基本上我們會有 `Issuers/Certificate` 基本元件，同時為了滿足 `DNS-01`, 這邊會需要一個額外的 `kubernetes secret` 供 `Issuers` 使用。此外 `Certificate` 最後則會產生一個 `kubernetes secret` 的合法憑證。

![Imgur](https://i.imgur.com/kMj495o.png)

## Cert-Manager
安裝 `Cert-Manager` 的部分有兩種安裝方式，第一種就是最基本也是最熟悉的，根據自己的環境去部屬事先提供好了 `Yaml` 檔案。
這部分可以參考[官方 Github ](https://github.com/jetstack/cert-manager/tree/master/contrib/manifests/cert-manager) 上面的檔案，基本上會根據叢集是否有啟用 `RBAC (Role-Based Access Control)` 來決定要部屬那個檔案。

另外一個安裝方式則是透過 `Package` 的概念來安裝，這邊採用的是 `Helm` 的方式來安裝，透過 `Helm` 的打包及管理，套件管理員將該服務所需要的 `Yaml` 都收集完畢並且打包，使用者只需要透過 `Helm` 的指令很簡單輕鬆的直接安裝完畢。

這邊我就直接使用 `Helm` 的方式來安裝 `Cert-Manager`.
```bash=
hwchiu➜~» helm install \
    --name cert-manager \
    --namespace kube-system \
    stable/cert-manager
```

想要安裝 `Helm` 可以參考[官網的教學](https://docs.helm.sh/using_helm/)

安裝完畢後，可以透過 `kubectl` 的指令去確認是否需要的 `deployment` 有運行起來
```bash=
hwchiu➜~» kubectl -n kube-system get  deployment/cert-manager
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cert-manager   1         1         1            1           5d
```

## Issuers
當 `cert-manager` 正確安裝後，我們接下來要開始處理 `Issuers`, 再我的範例之中，我採用的是 `clusterIssuers` 來建立一個供所有 `namespace` 都可以使用的 `Issuers`.

此外，由於我要採用的是 `DNS-01` 的方式來驗證網域的擁有權，同時我的 `DNS` 提供商是 `CloudFlare`. 因此我必須要先創立一個 `kubernetes sercet` 讓我的 `Issuers` 有能力操控我的 `CloudFlar`.


### CloudFlare
這部分會依據你的 `DNS` 供應商有不同的做法，詳細的可以參考 [Cert-Manager Supported DNS01 Providers](http://docs.cert-manager.io/en/latest/reference/issuers/acme/dns01.html#supported-dns01-providers)

1. 以 `CloudFlare` 爲範例，首先到你網域的控制台中去取得你的 `Public Key`. 可參考下圖
2. 得到該 `Public Key` 後，接下來要轉成 `Base64` 並且放到 `kubernetes sercet`.

![Imgur](https://i.imgur.com/niybCcQ.png)

假設該 `Public Key` 是 `12345678910`, 執行下列函式得到對應的 `Base64` 編碼結果
```bash=
echo -n "12345678910" | base64
```

接下來我們要透過 `yaml` 的方式創建 `kubernetes secret`.
打開文件 `sercet.yml` 並且貼上下列內容
```yaml=
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare
  namespace: kube-system
data:
  api: MTIzNDU2Nzg5MTA=
```

創建好上述的 `yaml` 檔案後，我們就透過 `kubectl create -f secret.yml`

### ClusterIssuers
接下來我們就要來創建我們的 `ClusterIssuers` 了，

這邊我們直接來看相關的 `yaml` 內容，有一些東西要注意
1. 我們在 `Issuer` 內目前使用了 `acme` 的方式來認證
2. server 的部分，一般來說會先用 `staging` 的內容來確認所有的資訊都正常且可以運作，正式環境中就會切換成 `v02` 的版本來使用
3. 於 `acme` 裡面，我們使用 `dns01` 的方式來驗證
    - 我們採用的是 `cloudflare` 的方式， 這邊就是填寫你擁有網域的信箱
    - 接下來要使用剛剛創立的 `kubernetes secret`, `name/key` 就對應到 `kubernetes secret` 內的 `metadata:name` 以及 `data/api`.

```yaml=
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: cert-demo
  namespace: default
spec:
  acme:
    #server: https://acme-v02.api.letsencrypt.org/directory
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your@mail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: cert-demo
    # ACME DNS-01 provider configurations
    dns01:
      # Here we define a list of DNS-01 providers that can solve DNS challenges
      providers:
        - name: cf-dns
          cloudflare:
            email: your@mail.com
            # A secretKeyRef to a cloudflare api key
            apiKeySecretRef:
              name: cloudflare
              key: api
```

創建上述 `ClusterIssuers` 的檔案之後，可以透過 `kubectl describe clusgerissuer/cert-demo` 去確認狀態，譬如
```bash=
.....
Status:
  Acme:
    Uri:  https://acme-staging-v02.api.letsencrypt.org/acme/acct/7037688
  Conditions:
    Last Transition Time:  2018-09-30T16:51:03Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

確認這邊是 `True/Ready` 之後, 對於 `Issuers` 的事情就告一段落了

## Certificates
正式進入 `Yaml` 之前，有幾個重要的事情要先注意
1. 每個 `Certificate` 可以設定 ` Subject Alternative Names(SANs)`, 亦即可以設定多個 `domain name`.
2. 針對設定的所有 `domain name`, `Issuers` 都必須要去處理。實際上在 `Issuers` 的設定中，可以設定多種方法，甚至是同時支援多個 `DNS` 供應商。
3. 因此對應`Certificae` 中的每個 `domain name`, 都要指派對應 `Issuers` 的可以用來處理該 `Certificate` 的物件，譬如 `dns-01` 內的某種 `dns` 供應商。
4. 每個 `Certificate` 最後會把得到的憑證資訊都寫入到 `Kubernetes Secret` 的資源內，所以我們必須要設定我們期望的名稱

有了上述的概念後，我們就可以來看一下 `Yaml` 的內容了
```yaml=
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: demo-certi
spec:
  secretName: demo-certi-tls
  issuerRef:
    name: cert-demo
    kind: ClusterIssuer
  dnsNames:
  - test.hwchiu.com
  acme:
    config:
    - dns01:
        provider: cf-dns
      domains:
      - test.hwchiu.com
```

這個範例中，我們要先指定該 `Certificate`  要使用的 `Issuers`. 這部分要透過 `issuerRef` 來指定剛剛創立的 `ClusterIssuers`.
接下來透過 `secretName` 去指定之後創立的 `kubernetes secret` 的名稱

最後要處理子域名的問題，在這個範例中只有一個，也就是 `test.hwchiu.com`.
所以在 `acme` 的部分就直接使用先前創立的 `ClusterIssuers` 內所描述的 `DNS01` 方式去進行認證

當這一切都準備就緒後就透過 `kubectl apply` 給部屬到叢集內。

接下來我們可以觀察整體運行的狀況，我們透過 `kubctl descriBe certificate demo-creti` 可以看一下 `certificae` 當前的狀態。

一個運行的範例如下
```bash=
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  CreateOrder   16s   cert-manager  Created new ACME order, attempting validation...
  Normal  IssueCert     15s   cert-manager  Issuing certificate...
  Normal  CertObtained  13s   cert-manager  Obtained certificate from ACME server
  Normal  CertIssued    13s   cert-manager  Certificate issued successfully
```

當一切都完畢後，就可以透過 `kubernetes get secret` 看到 `cert-manager` 已經幫忙創立了一個 `kubernetes secret`，裡面已經包含了你需要的憑證。
如果有使用 `ingress` 相關的，可以直接在 `Ingress` 那邊透過 `tls` 的設定直接使用該 `secret` 幫你的 `Ingress` 套上這個憑證囉

# Summary
使用任何的解決方案前，都要先思考一下到底自己遇到的是什麼問題，再從這個問題延伸去尋找方法來處理。接下來為了要更能夠熟悉該方法所帶來的好處以及壞處，就必須要花時間去瞭解其架構以及原理，最後透過動手操作的方式來實際體驗是否真的能夠解決問題。

本文的出發點很簡單，就是想要有一個解決方案能夠幫忙處理 `Ingress` 的憑證，然後因為沒錢所以希望使用 `Let's Encrypt` 的服務，但是礙於該服務所簽署的憑證有效期限很短，所以需要一直更新。

基於這個問題的情況下，我開始研究了 `Cert-Manager`，並且透過一系列的操作與體驗來理解如何使用 `Cert-Manager` 解決我的問題。

最後也希望大家都能夠保持類似的態度與想法來解決每一個問題，藉由這些步驟能夠更深刻的體會每個解決方案背後的原理以及設計思緒，將這些概念融入自己的思想之中以不停的成長學習。

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/mLwKb3zrx

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4XNG

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