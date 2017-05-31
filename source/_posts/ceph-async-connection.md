---
title: Ceph Network - AsyncConnection
keywords: 'Network,Ceph,SDS,SourceCode,Linux,ScaelOutStorage'
date: 2017-05-31 06:50:49
tags:
	- Network
	- Ceph
	- SDS
	- SourceCode
	- Linux
---

AsyncConnection 此物件代表整個 connection，裡面提供了收送(Write/Read)兩個主要介面供應用層(OSD/MON等)使用外，裡面也處理了整個 **Ceph Node**收送封包的邏輯處理，這部分比較像是一個 **finite state machine(FSM)**，當前狀態是什麼時候，收到的封包是什麼，就切換到什麼狀態來處理。
每個 AsyncConnection 會像底層的 **Event Engine**註冊一個 **call back function**，當該 **connection** 接收到封包後，就會觸發該 **function**，而此 **function**就是 **Process**，所以接下來會從此 function 當作下手點，主要研究雙方連線建立的過程，特別是從 **Service side** 去觀察 **Accept** 封包後的流程。


<!--more-->

Process
-------
- 跑超巨大迴圈，值到狀態穩定 (prev_state == state)
- 如果不符合大部分的 State，則呼叫 `_process_connection` 來處理，譬如 (`STATE_ACCEPTING`)

#### STATE_ACCEPTING
- 因為先前切換到此狀態時，是透過 external event 呼叫的(只會執行一次)，所以這邊要將該 readable event handler (process) 正式的丟給 event center 一次
- 發送 banner
- 發送當前的 addr + port
- 呼叫 try_send 去發送訊息
- 若成功 (r == 0), 狀態切換到 STATE_ACCEPTING_WAIT_BANNER_ADDR
- 若失敗 (r > 0),狀態切換到 STATE_WAIT_SEND，並且使用一個 strate_after_send 的變數來記住當成功送出後要切換成什麼狀態
- 若失敗 (r < 0),真的失敗了QQ

#### STATE_ACCEPTING_WAIT_BANNER_ADDR
- 讀取對方的 banner + addr(ip/port)
- 比較 banner 資訊
- 若對方不知道自己的 addr，則透過 socket 的資訊取得並且記錄下來
- 狀態改成 STATE_ACCEPTING_WAIT_CONNECT_MSG

#### STATE_ACCEPTING_WAIT_CONNECT_MSG
- 讀取 connect_msg 大小的資料
``` c
0099 struct ceph_msg_connect {
0100     __le64 features;     /* supported feature bits */
0101     __le32 host_type;    /* CEPH_ENTITY_TYPE_* */
0102     __le32 global_seq;   /* count connections initiated by this host */
0103     __le32 connect_seq;  /* count connections initiated in this session */
0104     __le32 protocol_version;
0105     __le32 authorizer_protocol;
0106     __le32 authorizer_len;
0107     __u8  flags;         /* CEPH_MSG_CONNECT_* */
0108 } __attribute__ ((packed));
```
- 切換到 STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH

#### STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH
- 根據之前讀取到的 connect_msg 來操作
- 如果對方有設定 authorizer_len 的話，則在額外讀取 authorizer 相關的資訊
- 設定 peer 的 host type
- 根據 host type，取得對應的 policy
- 呼叫 handle_connect_msg 處理該 connection_msg
#### STATE_ACCEPTING_WAIT_SEQ
- 從對面讀取其使用的 seq
- 呼叫 discard_requeued_up_to 來處理，根據當前收到的 seq 來做條件
    - 將 out_q 一些不符合條件的成員都移除
- 狀態改成 STATE_ACCEPTING_READY
#### STATE_ACCEPTING_READY
- 清空 connect_msg
- 狀態改成 STATE_OPEN
- 如果當前 queue 內有資料(也許是先前 existing connection產生的?)，馬上送一個 write_handler 將其處理完畢
#### STATE_OPEN
- 讀取 TAG，根據 TAG 不同的數值執行不同的事情
    - CEPH_MSGR_TAG_MSG: 代表有訊息近來，故將狀態切換成 STATE_OPEN_MESSAGE_HEADER
#### STATE_OPEN_MESSAGE_HEADER
- 根據 feature 的值，決定要走新版還是舊版的 header
```c++
0443           if (has_feature(CEPH_FEATURE_NOSRCADDR))
0444             len = sizeof(header);
0445           else
0446             len = sizeof(oldheader);
```
- 讀取 header 大小的資料，然後將需要的資料都抓出來記錄下來。
- 驗證對方送來資料的 CRC 是否正確
- 將相關資料給 reset (這些結構都跟 header 有關)
    - data_buf
    - front
    - middle
    - data
- 記錄收到時間的時間戳
``` c++
recv_stamp = ceph_clock_now();
```
- 將狀態改變成 STATE_OPEN_MESSAGE_THROTTLE_MESSAGE

#### STATE_OPEN_MESSAGE_THROTTLE_MESSAGE
- 在 Policy 中有兩個關於 Throttle 的變數
``` c++
0085     /**
0086      *  The throttler is used to limit how much data is held by Messages from
0087      *  the associated Connection(s). When reading in a new Message, the Messenger
0088      *  will call throttler->throttle() for the size of the new Message.
0089      */
0090     Throttle *throttler_bytes;
0091     Throttle *throttler_messages;
```
- 這個 function 檢查是否有 throttle 訊息的數量限制，若有限制且超過上限，則建立一個 time event，並儲存下來。
- 最後將狀態切換到 STATE_OPEN_MESSAGE_THROTTLE_BYTES

#### STATE_OPEN_MESSAGE_THROTTLE_BYTES;
- 從 connection 讀取資料，分別對應到 header 中的三個成員
    - front
    - middle
    - data
- 此 function 則是檢查 throttle 訊息的 bytes 數量，若數量超過上限，也是建議一個 time event並存下來，待之後處理
- 最後將狀態切換到 STATE_OPEN_MESSAGE_THROTTLE_DISPATCH_QUEUE

#### STATE_OPEN_MESSAGE_THROTTLE_DISPATCH_QUEUE
- 如果剛剛有在 **STATE_OPEN_MESSAGE_THROTTLE_BYTES** 讀取到 front/middle/data 的資料的話，則這邊要確認 disaptch 本身的 throttle 有沒有超過，若超過也是送一個 time event 待稍後再來重新試試看
- 紀錄 throttle 的時間戳
- 狀態切換成 STATE_OPEN_MESSAGE_READ_FRONT
``` c++
0561           throttle_stamp = ceph_clock_now();
0562           state = STATE_OPEN_MESSAGE_READ_FRONT;
```

#### STATE_OPEN_MESSAGE_READ_FRONT
- 讀取前段資料，將內容先放到本身的 front 變數中
- 將狀態切成 STATE_OPEN_MESSAGE_READ_MIDDLE

#### STATE_OPEN_MESSAGE_READ_MIDDLE
- 讀取中段資料，將內容先放到本身的 middle 變數中
- 將狀態切成 STATE_OPEN_MESSAGE_READ_DATA_PREPARE

#### STATE_OPEN_MESSAGE_READ_DATA_PREPARE
- 準備好 buffer 供之後讀取 data 用，其中 data 部分除了長度外，還有 offset 也要處理
- 將狀態切成 STATE_OPEN_MESSAGE_READ_DATA

#### STATE_OPEN_MESSAGE_READ_DATA
- 透過一個迴圈嘗試將資料讀取出來並且放到 data 內
- 最後狀態切換到 STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH

#### STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH
- 跟 header 一樣，根據 feature 決定使用新舊版本的 footer 格式
- 讀取 footer 的資料，如各區段的CRC等
- 透過 decode_message 此 function，將收集到的 (front, middle, data, footer.etc) 組合成一個完整的 message 格是的封包
```c++
0270 Message *decode_message(CephContext *cct, int crcflags,
0271                         ceph_msg_header& header,
0272                         ceph_msg_footer& footer,
0273                         bufferlist& front, bufferlist& middle,
0274                         bufferlist& data)
....
0315   // make message
0316   Message *m = 0;
0317   int type = header.type;
0318   switch (type) {
0319 
0320     // -- with payload --
0321 
0322   case MSG_PGSTATS:
0323     m = new MPGStats;
0324     break;
0325   case MSG_PGSTATSACK:
0326     m = new MPGStatsAck;
0327     break;
0328 
0329   case CEPH_MSG_STATFS:
0330     m = new MStatfs;
0331     break;
0332   case CEPH_MSG_STATFS_REPLY:
0333     m = new MStatfsReply;
0334     break;
0335   case MSG_GETPOOLSTATS:
0336     m = new MGetPoolStats;
0337     break;
0338   case MSG_GETPOOLSTATSREPLY:
0339     m = new MGetPoolStatsReply;
...
```
- 針對該 message 設定一些相關屬性
    - byte_throttler
    - message_throttler
    - dispatch_throttle_size
    - recv_stamp
    - throttle_stamp
    - recv_complete_stamp
- 針對 sequence 進行一些判斷，當前的 message 可能中間有遺漏，或是很久以前的 message
- 將該 messaged 的 sequence 當作目前最後一個收到 sequence
``` c++
0765           // note last received message.
0766           in_seq.set(message->get_seq());
```
- 將狀態改成 STATE_OPEN
- 將本訊息塞入到 dispatch_queue 內，供應用層去處理

#### STATE_CONNECTING
- 檢查 cs 此變數，如果之前有連線過，則關閉先前的連線
    - 同時也先刪除之前的 event
- 呼叫 worker 跟對方的 socket 去連線
- 創造一個 read_handler 的 event，來處理接下來收到封包的行為
    - read_handler 就是 process
- 狀態切換成 STATE_CONNECTING_RE

#### STATE_CONNECTING_RE
- 檢查當前 connectionSocket 連線狀態，本身會若發現沒有連線則會自己重新連線
    - r < 0, 連線依然失敗，則判定有問題， goto 離開
    - r == 1, 成功，不做事情
    - r == 0, 重連過程中有出現錯誤，可能是 EINPROGRESS  或是 EALREADY。
```c++
0055   int is_connected() override {
0056     if (connected)
0057       return 1;
0058 
0059     int r = handler.reconnect(sa, _fd);
0060     if (r == 0) {
0061       connected = true;
0062       return 1;
0063     } else if (r < 0) {
0064       return r;
0065     } else {
0066       return 0;
0067     }
0068   }
```
``` c++
0211 int NetHandler::reconnect(const entity_addr_t &addr, int sd)
0212 {
0213   int ret = ::connect(sd, addr.get_sockaddr(), addr.get_sockaddr_len());
0214 
0215   if (ret < 0 && errno != EISCONN) {
0216     ldout(cct, 10) << __func__ << " reconnect: " << strerror(errno) << dendl;
0217     if (errno == EINPROGRESS || errno == EALREADY)
0218       return 1;
0219     return -errno;
0220   }
0221 
0222   return 0;
0223 }
```
- 嘗試送出 CEPH_BANNER
- 若成功，狀態切換成 STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY
- 若失敗，狀態切換成 STATE_WAIT_SEND，待之後重送後再處理。

#### STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY
- 讀取 SERVER 端送來的 ceph_banner
- 讀取 SERVER 端送來的 address * 2
    - server + server 端看到的 client
```c++
0960         try {
0961           ::decode(paddr, p);
0962           ::decode(peer_addr_for_me, p);
0963         } catch (const buffer::error& e) {
0964           lderr(async_msgr->cct) << __func__ <<  " decode peer addr failed " << dendl;
0965           goto fail;
0966         }
```
- 比較兩邊的 CEPH_BANNER
- 比較 peer addr (server address)
    - 我自己 socket 看到的
    - 對方送過來的
- 將自己的 address 送給 server
- 若成功，將狀態切換成 STATE_CONNECTING_SEND_CONNECT_MSG
- 若失敗，將狀態切換成 STATE_WAIT_SEND，之後再處理。

#### STATE_CONNECTING_SEND_CONNECT_MSG
- 設定 connect_msg 的資訊
- 嘗試送給 server
- 若成功則將狀態切換到 STATE_CONNECTING_WAIT_CONNECT_REPLY

#### STATE_CONNECTING_WAIT_CONNECT_REPLY
- 讀取 server回傳回來的 ceph_msg_connect_reply
``` c++
0110 struct ceph_msg_connect_reply {
0111     __u8 tag;
0112     __le64 features;     /* feature bits for this session */
0113     __le32 global_seq;
0114     __le32 connect_seq;
0115     __le32 protocol_version;
0116     __le32 authorizer_len;
0117     __u8 flags;
0118 } __attribute__ ((packed));
```
- 狀態切換成 STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH


#### STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH
- 根據來回資料中 authorizer 來判斷要不要進行相關處理
- 呼叫 `handle_connect_reply` 進行處理
- 上述處理完畢後，狀態必須要改變，若沒有變則透過 assert 處理

#### STATE_CONNECTING_READY
- 萬歲!
``` c++
1163     case STATE_CONNECTING_READY:
1164       {
1165         // hooray!
1166         peer_global_seq = connect_reply.global_seq;
```
- 對 dispatch_queue 設定當前 connection
    - dispatch_queue 這邊有兩種類型，一種是存放 message，一種則是 Type + Connection，這邊屬於第二種
    - 在 dispatch_queue 的 loop 中，會針對這兩種去處理，若是 messag 的，則直接將此訊息丟給所有註冊的 dispatcher，反之則根據 type 執行不同的任務
    - 這邊放入的是 D_CONNECT 的 event，所以之後會執行 `ms_deliver_handle_connect` 這支 function。
    - 接者這支 function 則是會通知所有 dispathcer 目前有新的連線到來，呼叫對應的 `ms_handle_connect`來處理
```c++
0155     while (!mqueue.empty()) {
0156       QueueItem qitem = mqueue.dequeue();
0157       if (!qitem.is_code())
0158         remove_arrival(qitem.get_message());
0159       lock.Unlock();
0160 
0161       if (qitem.is_code()) {
0162         if (cct->_conf->ms_inject_internal_delays &&
0163             cct->_conf->ms_inject_delay_probability &&
0164             (rand() % 10000)/10000.0 < cct->_conf->ms_inject_delay_probability) {
0165           utime_t t;
0166           t.set_from_double(cct->_conf->ms_inject_internal_delays);
0167           ldout(cct, 1) << "DispatchQueue::entry  inject delay of " << t
0168                         << dendl;
0169           t.sleep();
0170         }
0171         switch (qitem.get_code()) {
0172         case D_BAD_REMOTE_RESET:
0173           msgr->ms_deliver_handle_remote_reset(qitem.get_connection());
0174           break;
0175         case D_CONNECT:
0176           msgr->ms_deliver_handle_connect(qitem.get_connection());
0177           break;
0178         case D_ACCEPT:
0179           msgr->ms_deliver_handle_accept(qitem.get_connection());
0180           break;
0181         case D_BAD_RESET:
0182           msgr->ms_deliver_handle_reset(qitem.get_connection());
0183           break;
0184         case D_CONN_REFUSED:
0185           msgr->ms_deliver_handle_refused(qitem.get_connection());
0186           break;
0187         default:
0188           ceph_abort();
0189         }
0190       } else {
0191         Message *m = qitem.get_message();
0192         if (stop) {
0193           ldout(cct,10) << " stop flag set, discarding " << m << " " << *m << dendl;
0194           m->put();
0195         } else {
0196           uint64_t msize = pre_dispatch(m);
0197           msgr->ms_deliver_dispatch(m);
0198           post_dispatch(m, msize);
0199         }

```
``` c++
0610   /**
0611    * Notify each Dispatcher of a new Connection. Call
0612    * this function whenever a new Connection is initiated or
0613    * reconnects.
0614    *
0615    * @param con Pointer to the new Connection.
0616    */
0617   void ms_deliver_handle_connect(Connection *con) {
0618     for (list<Dispatcher*>::iterator p = dispatchers.begin();
0619          p != dispatchers.end();
0620          ++p)
0621       (*p)->ms_handle_connect(con);
0622   }
```

- 呼叫 AsyncMessegner 內的 ms_deliver_handle_fast_connect
    - fast_connect 相對於 connect 是更早會處理的函式，底層可確保此 function 一定會在有任何 message 被處理前先呼叫。
```c++
0110   /**
0111    * This function will be called synchronously whenever a Connection is
0112    * newly-created or reconnects in the Messenger, if you support fast
0113    * dispatch. It is guaranteed to be called before any messages are
0114    * dispatched.
0115    *
0116    * @param con The new Connection which has been established. You are not
0117    * granted a reference to it -- take one if you need one!
0118    */
0119   virtual void ms_handle_fast_connect(Connection *con) {}
```
- 如果當前 queue 內有訊息，這時候再發送一個外部的 write_handler把queue給清空。
    - 可能是由於先前的 try_send 沒有成功
- 到這邊後，連線就完成了，可以開始供應用層各種發送訊息了。

Accept
------
- 將 socket 跟 addr 都記錄下來，並且將狀態改成 **STATE_ACCEPTING**
- 發送一個 external event(read_handler) 給 eventCenter
    - 此 read_handler 就是 `process`，會一直根據當前 state 的狀態來進行各種處理
    
handle_connect_reply
- 根據 reply 內的 tag 類型來執行各種不同事情, 大部分都是錯誤相關的處理，若一切都正常的話，則會是`CEPH_MSGR_TAG_READY`，此時會將狀態切換成 STATE_CONNECTING_READY
    - CEPH_MSGR_TAG_FEATURES
    - CEPH_MSGR_TAG_BADPROTOVER
    - CEPH_MSGR_TAG_BADAUTHORIZER
    - CEPH_MSGR_TAG_RESETSESSION
    - CEPH_MSGR_TAG_RETRY_GLOBAL
    - CEPH_MSGR_TAG_RETRY_SESSION
    - CEPH_MSGR_TAG_WAIT
    - CEPH_MSGR_TAG_SEQ
    - CEPH_MSGR_TAG_READY



handle_connect_msg
------------------
- 根據 peer type 取得對應的 proto_version，放到 ceph_msg_connect_reply 的變數中
```c++
0671       case CEPH_ENTITY_TYPE_OSD: return CEPH_OSDC_PROTOCOL;
0672       case CEPH_ENTITY_TYPE_MDS: return CEPH_MDSC_PROTOCOL;
0673       case CEPH_ENTITY_TYPE_MON: return CEPH_MONC_PROTOCOL;
```
- 若兩邊的 proto_version 不一致，則呼叫 `_reply_aceept` 去處理。
- 若對方有要使用 cephX
    - 根據不同的 protocol type (OSD/MDS/MOM) 進行不同的處理
- 檢查兩邊的 feature set 是否滿足彼此，若有問題則呼叫 `_reply_accept` 去處理
- 進行用戶驗證，失敗則呼叫 `_reply_accept`
- 若以前 peer addr 曾經有 connection 存在過，這時候就要進行一些處理，主要的處理都是基於兩個變數來決定，global_seq 以及 connect_seq
    - global_seq 代表的是這個host已經建立過多少條 connection
    - connect_seq 代表的是這個 session建立過多少條 connection
- 某些情況下，會嘗試捨棄舊有的 connection 並建立新的 connection 來使用
- 某些情況則是會繼續使用舊有的 connection，然後把一些新的資訊賦予到舊有 connection 的成員中
- 呼叫 accept_conn 將連線給記錄下來放到 conns 中，並且從 accepting_conns 中移除，
- 最後則將狀態改成 STATE_ACCEPTING_WAIT_SEQ


Connect
-------
- 當 AsyncMessager 創立 AsyncConnection時，就會先呼叫此 function 進行連線了，後續若有訊息發送時，會透過 `_connect`重新連線。
- 設定 peer 的 addr以及 policy
- 呼叫 _connect 完成最後的連線步驟
``` c++
0200   void connect(const entity_addr_t& addr, int type) {
0201     set_peer_type(type);
0202     set_peer_addr(addr);
0203     policy = msgr->get_policy(type);
0204     _connect();
0205   }
```

Summary
-------
此 **AsyncConnection** 內容眾多，目前先主要觀察到整個建立連線的步驟，包含了 **Accept** 以及 **connect** 這兩方面，不過在 **connect** 方面就比較少著墨，基本上是跟 **Accept**有種對稱的關係，畢竟有人收封包，就要有人送封包。
有機會再來把 **read/write** 相關的介面也都看一次，到時候可以更瞭解整體收送的行為。


