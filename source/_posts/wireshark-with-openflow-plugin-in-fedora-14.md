---
title: Wireshark with Openflow-Plugin in Fedora 14
keywords: 'wireshark,openflow,extenstion'
date: '2013-08-18 05:14'
comments: true
tags:
  - SDN
  - Openflow
  - Network
  - Wireshark
abbrlink: 59436
description: 這篇文章主要分享如何於 Wireshark 中安裝額外的模組使得其有能力去解析 OpenFlow 的封包結構，對於研究 Openflow 的人來說這是一個很好使用的工具，能夠觀察 Switch to Controller, Controller to Switch 等各種封包.

---

# Preface

參考這篇文章
[http://networkstatic.net/installing-wireshark-on-linux-for-openflow-packet-captures/](http://networkstatic.net/installing-wireshark-on-linux-for-openflow-packet-captures/)

# 安裝wireshark source
```bash=
- wget http://wiresharkdownloads.riverbed.com/wireshark/src/wireshark-1.8.8.tar.bz2
(http://wiresharkdownloads.riverbed.com/wireshark/src/ 自己挑選一個版本下載)
- bunzip2 wireshark-1.8.8.tar.bz2
- tar -xvf wireshark-1.8.8.tar
- cd wireshark-1.8.8
- ./autogen.sh
- ./configure
- make
(這邊錯誤通常是少了某些套件，根據錯誤訊息再去安裝即可)
- make install
- sudo ldconfi
- ./wireshark
    ```


# 編譯openflow plugin
## Options
- hg clone https://bitbucket.org/barnstorm/of-dissector
- cd of-dissector/src
- apt-get install scons
- 修改 packet-openflow.c

``` c
Change from:
    static void dissect_dl_type(....)
    {
    ....
    	const char* description = try_val_to_str(dl_type, etype_vals);
    ....
    }
```


``` c
    To:
    static void dissect_dl_type(....)
    {
    ....
    	const char* description = match_strval(dl_type, etype_vals);
    ....
    }
```

```bash=
- scons install
- export WIRESHARK=/path_to_wireshark_source/
- scons install
- cp openflow.so /usr/lib/wireshark/libwireshark1/plugins/openflow.so
```

* Options 2
```bash=
- git clone git://openflow.org/openflow.git
- cd openflow
- ./boot.sh
- ./configure
- make
- sudo make install
- cd utilities/wireshark_dissectors/openflow
- 修改 packet-openflow.c
```

```
  Change from:
    void proto_reg_handoff_openflow()
    {
      openflow_handle = create_dissector_handle(dissect_openflow, proto_openflow);
      dissector_add(TCP_PORT_FILTER, global_openflow_proto, openflow_handle);
    }
```


``` c
    To:
    void proto_reg_handoff_openflow()
    {
      openflow_handle = create_dissector_handle(dissect_openflow, proto_openflow);
      dissector_add_uint(TCP_PORT_FILTER, global_openflow_proto, openflow_handle);
    }
```

# 安裝openflow plugin
- make ( pwd = utilities/wireshark_dissectors/openflow)
- make install

# Use
開啟wireshark即可觀看openflow protocol囉

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