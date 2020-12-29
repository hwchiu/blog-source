---
title: Ceph Network - AsyncConnection
keywords: 'Network,Ceph,SDS,SourceCode,Linux,ScaelOutStorage'
tags:
  - Network
  - Ceph
  - SDS
  - SourceCode
  - Linux
abbrlink: 6732
date: 2017-05-31 06:50:49
---


AsyncConnection 此物件代表整個 connection，裡面提供了收送(Write/Read)兩個主要介面供應用層(OSD/MON等)使用外，裡面也處理了整個 **Ceph Node**收送封包的邏輯處理，這部分比較像是一個 **finite state machine(FSM)**，當前狀態是什麼時候，收到的封包是什麼，就切換到什麼狀態來處理。
每個 AsyncConnection 會像底層的 **Event Engine**註冊一個 **call back function**，當該 **connection** 接收到封包後，就會觸發該 **function**，而此 **function**就是 **Process**，所以接下來會從此 function 當作下手點，主要研究雙方連線建立的過程，特別是從 **Service side** 去觀察 **Accept** 封包後的流程。


<!--more-->


# Accept
- 當 **Server Side** 呼叫此 **function**後，就會開始等待 **client**來連線。
- 將 socket 跟 addr 都記錄下來，並且將狀態改成 **STATE_ACCEPTING**
- 發送一個 external event(read_handler) 給 eventCenter
    - 此 read_handler 就是 `process`，會一直根據當前 state 的狀態來進行各種處理

```c++
1866 void AsyncConnection::accept(ConnectedSocket socket, entity_addr_t &addr)
1867 {
1868   ldout(async_msgr->cct, 10) << __func__ << " sd=" << socket.fd() << dendl;
1869   assert(socket.fd() >= 0);
1870
1871   std::lock_guard<std::mutex> l(lock);
1872   cs = std::move(socket);
1873   socket_addr = addr;
1874   state = STATE_ACCEPTING;
1875   // rescheduler connection in order to avoid lock dep
1876   center->dispatch_event_external(read_handler);
1877 }
```

# Process
當**AsyncConnection**有收到任何封包時，就會呼叫這個 **function**，我們這邊假設我們是 **Server Side**，然後當前 **Socket** 處於一種 **Accepting** 的狀態，在此狀態下收到連線的封包後，會怎麼處理。

``` c++
0329 void AsyncConnection::process()
0330 {
0331   ssize_t r = 0;
0332   int prev_state = state;
0333 #if defined(WITH_LTTNG) && defined(WITH_EVENTTRACE)
0334   utime_t ltt_recv_stamp = ceph_clock_now();
0335 #endif
0336   bool need_dispatch_writer = false;
0337   std::lock_guard<std::mutex> l(lock);
0338   last_active = ceph::coarse_mono_clock::now();
0339   do {
0340     ldout(async_msgr->cct, 20) << __func__ << " prev state is " << get_state_name(prev_state) << dendl;
0341     prev_state = state;
0342     switch (state) {
0343       case STATE_OPEN:
....
0837       default:
0838         {
0839           if (_process_connection() < 0)
0840             goto fail;
0841           break;
0842         }
0843     }
0844   } while (prev_state != state);
0845
0846   if (need_dispatch_writer && is_connected())
0847     center->dispatch_event_external(write_handler);
0848   return;
0849
0850  fail:
0851   fault();
0852 }
```
基本上就是跑一個大迴圈，根據當前狀態處理不同的事情，直到狀態已經穩定 **(prev_state == state)**， 如果不符合大部分的 State，則呼叫 `_process_connection` 來處理，譬如 **STATE_ACCEPTING**。
一旦建立 **Server Socket**時並且開始透過 **Accept** 等待連線時，狀態則初始化為 **STATE_ACCEPTING**，所以接下來就從這個狀態開始往下研究。

## STATE_ACCEPTING
- 因為先前切換到此狀態時，是透過 **external event** 呼叫的(只會執行一次)，所以這邊要將該 **readable event handler (process)** 正式的丟給 **event center** 一次
- 接下來要發送一些訊息到對面去，這邊需要下列資訊
    - CEPH_BANNER
    - Addr + Port
- 呼叫 try_send 去發送訊息
    - 若成功 (r == 0), 狀態切換到 **STATE_ACCEPTING_WAIT_BANNER_ADDR**
    - 若失敗 (r > 0),狀態切換到 **STATE_WAIT_SEND**，並且使用一個 **strate_after_send** 的變數來記住當成功送出後要切換成什麼狀態
    - 若失敗 (r < 0),真的失敗了，就當作失敗處理。

```c++
1215     case STATE_ACCEPTING:
1216       {
1217         bufferlist bl;
1218         center->create_file_event(cs.fd(), EVENT_READABLE, read_handler);
1219
1220         bl.append(CEPH_BANNER, strlen(CEPH_BANNER));
1221
1222         ::encode(async_msgr->get_myaddr(), bl, 0); // legacy
1223         port = async_msgr->get_myaddr().get_port();
1224         ::encode(socket_addr, bl, 0); // legacy
1225         ldout(async_msgr->cct, 1) << __func__ << " sd=" << cs.fd() << " " << socket_addr << dendl;
1226
1227         r = try_send(bl);
1228         if (r == 0) {
1229           state = STATE_ACCEPTING_WAIT_BANNER_ADDR;
1230           ldout(async_msgr->cct, 10) << __func__ << " write banner and addr done: "
1231             << get_peer_addr() << dendl;
1232         } else if (r > 0) {
1233           state = STATE_WAIT_SEND;
1234           state_after_send = STATE_ACCEPTING_WAIT_BANNER_ADDR;
1235           ldout(async_msgr->cct, 10) << __func__ << " wait for write banner and addr: "
1236                               << get_peer_addr() << dendl;
1237         } else {
1238           goto fail;
1239         }
1240
1241         break;
1242       }
```

## STATE_ACCEPTING_WAIT_BANNER_ADDR
- 讀取對方的封包的資訊(代表 Client Side 也必須要發送相關訊息)
    - CEPH_BANNER
    - Addr + Port
- 比較 banner 資訊
- 若對方不知道自己的 addr，則透過 socket 的資訊取得並且記錄下來
- 狀態改成 **STATE_ACCEPTING_WAIT_CONNECT_MSG**

```c++
1243     case STATE_ACCEPTING_WAIT_BANNER_ADDR:
1244       {
1245         bufferlist addr_bl;
1246         entity_addr_t peer_addr;
1247
1248         r = read_until(strlen(CEPH_BANNER) + sizeof(ceph_entity_addr), state_buffer);
1249         if (r < 0) {
1250           ldout(async_msgr->cct, 1) << __func__ << " read peer banner and addr failed" << dendl;
1251           goto fail;
1252         } else if (r > 0) {
1253           break;
1254         }
1255
1256         if (memcmp(state_buffer, CEPH_BANNER, strlen(CEPH_BANNER))) {
1257           ldout(async_msgr->cct, 1) << __func__ << " accept peer sent bad banner '" << state_buffer
1258                                     << "' (should be '" << CEPH_BANNER << "')" << dendl;
1259           goto fail;
1260         }
1261
1262         addr_bl.append(state_buffer+strlen(CEPH_BANNER), sizeof(ceph_entity_addr));
1263         {
1264           bufferlist::iterator ti = addr_bl.begin();
1265           ::decode(peer_addr, ti);
1266         }
1267
1268         ldout(async_msgr->cct, 10) << __func__ << " accept peer addr is " << peer_addr << dendl;
1269         if (peer_addr.is_blank_ip()) {
1270           // peer apparently doesn't know what ip they have; figure it out for them.
1271           int port = peer_addr.get_port();
1272           peer_addr.u = socket_addr.u;
1273           peer_addr.set_port(port);
1274           ldout(async_msgr->cct, 0) << __func__ << " accept peer addr is really " << peer_addr
1275                              << " (socket is " << socket_addr << ")" << dendl;
1276         }
1277         set_peer_addr(peer_addr);  // so that connection_state gets set up
1278         state = STATE_ACCEPTING_WAIT_CONNECT_MSG;
1279         break;
1280       }
```

## STATE_ACCEPTING_WAIT_CONNECT_MSG
- 讀取 connect_msg 大小的資料,並且存放到 **AsyncConnection** 的成員 **connect_msg**中，該結構如下，紀錄了如 **feature**, **type** 等資訊。
- 最後將狀態轉換成 **
**
``` c++
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

``` c++
1282     case STATE_ACCEPTING_WAIT_CONNECT_MSG:
1283       {
1284         r = read_until(sizeof(connect_msg), state_buffer);
1285         if (r < 0) {
1286           ldout(async_msgr->cct, 1) << __func__ << " read connect msg failed" << dendl;
1287           goto fail;
1288         } else if (r > 0) {
1289           break;
1290         }
1291
1292         connect_msg = *((ceph_msg_connect*)state_buffer);
1293         state = STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH;
1294         break;
1295       }
```


## STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH
- 根據之前讀取到的 **connect_msg** 來操作
- 如果對方有設定 authorizer_len 的話，則在額外讀取 authorizer 相關的資訊
- 設定 peer 的 host type
- 根據 host type，取得對應的 policy
- 呼叫 handle_connect_msg 處理該 connection_msg
- 最後確認狀態已經不是本來的 **STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH**
    - 狀態理論上要因為呼叫了 **handle_connect_msg** 而變換，正常來說要變成 **STATE_ACCEPTING_WAIT_SEQ**
```c++
1297     case STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH:
1298       {
1299         bufferlist authorizer_reply;
1300
1301         if (connect_msg.authorizer_len) {
1302           if (!authorizer_buf.length())
1303             authorizer_buf.push_back(buffer::create(connect_msg.authorizer_len));
1304
1305           r = read_until(connect_msg.authorizer_len, authorizer_buf.c_str());
1306           if (r < 0) {
1307             ldout(async_msgr->cct, 1) << __func__ << " read connect authorizer failed" << dendl;
1308             goto fail;
1309           } else if (r > 0) {
1310             break;
1311           }
1312         }
1313
1314         ldout(async_msgr->cct, 20) << __func__ << " accept got peer connect_seq "
1315                              << connect_msg.connect_seq << " global_seq "
1316                              << connect_msg.global_seq << dendl;
1317         set_peer_type(connect_msg.host_type);
1318         policy = async_msgr->get_policy(connect_msg.host_type);
1319         ldout(async_msgr->cct, 10) << __func__ << " accept of host_type " << connect_msg.host_type
1320                                    << ", policy.lossy=" << policy.lossy << " policy.server="
1321                                    << policy.server << " policy.standby=" << policy.standby
1322                                    << " policy.resetcheck=" << policy.resetcheck << dendl;
1323
1324         r = handle_connect_msg(connect_msg, authorizer_buf, authorizer_reply);
1325         if (r < 0)
1326           goto fail;
1327
1328         // state is changed by "handle_connect_msg"
1329         assert(state != STATE_ACCEPTING_WAIT_CONNECT_MSG_AUTH);
1330         break;
1331       }
```


## STATE_ACCEPTING_WAIT_SEQ
- 從對面讀取其使用的 seq
- 呼叫 discard_requeued_up_to 來處理，根據當前收到的 seq 來做條件
    - 將 **out_q** 一些不符合條件的成員都移除
- 狀態改成 **STATE_ACCEPTING_READY**
-
```c++
1333     case STATE_ACCEPTING_WAIT_SEQ:
1334       {
1335         uint64_t newly_acked_seq;
1336         r = read_until(sizeof(newly_acked_seq), state_buffer);
1337         if (r < 0) {
1338           ldout(async_msgr->cct, 1) << __func__ << " read ack seq failed" << dendl;
1339           goto fail_registered;
1340         } else if (r > 0) {
1341           break;
1342         }
1343
1344         newly_acked_seq = *((uint64_t*)state_buffer);
1345         ldout(async_msgr->cct, 2) << __func__ << " accept get newly_acked_seq " << newly_acked_seq << dendl;
1346         discard_requeued_up_to(newly_acked_seq);
1347         state = STATE_ACCEPTING_READY;
1348         break;
1349       }
```

## STATE_ACCEPTING_READY
- 清空 **connect_msg**
- 狀態改成 **STATE_OPEN**
- 如果當前 queue 內有資料(也許是先前 existing connection產生的?)，馬上送一個 write_handler 將其處理完畢

```c++
1351     case STATE_ACCEPTING_READY:
1352       {
1353         ldout(async_msgr->cct, 20) << __func__ << " accept done" << dendl;
1354         state = STATE_OPEN;
1355         memset(&connect_msg, 0, sizeof(connect_msg));
1356
1357         if (delay_state)
1358           assert(delay_state->ready());
1359         // make sure no pending tick timer
1360         if (last_tick_id)
1361           center->delete_time_event(last_tick_id);
1362         last_tick_id = center->create_time_event(inactive_timeout_us, tick_handler);
1363
1364         write_lock.lock();
1365         can_write = WriteStatus::CANWRITE;
1366         if (is_queued())
1367           center->dispatch_event_external(write_handler);
1368         write_lock.unlock();
1369         maybe_start_delay_thread();
1370         break;
1371       }
```

## STATE_OPEN
- 讀取 TAG，根據 TAG 不同的數值執行不同的事情
    - CEPH_MSGR_TAG_MSG: 代表有訊息近來，故將狀態切換成 STATE_OPEN_MESSAGE_HEADER
```c++
0343       case STATE_OPEN:
0344         {
0345           char tag = -1;
0346           r = read_until(sizeof(tag), &tag);
0347           if (r < 0) {
0348             ldout(async_msgr->cct, 1) << __func__ << " read tag failed" << dendl;
0349             goto fail;
0350           } else if (r > 0) {
0351             break;
0352           }
0353
0354           if (tag == CEPH_MSGR_TAG_KEEPALIVE) {
0355             ldout(async_msgr->cct, 20) << __func__ << " got KEEPALIVE" << dendl;
0356             set_last_keepalive(ceph_clock_now());
0357           } else if (tag == CEPH_MSGR_TAG_KEEPALIVE2) {
0358             state = STATE_OPEN_KEEPALIVE2;
0359           } else if (tag == CEPH_MSGR_TAG_KEEPALIVE2_ACK) {
0360             state = STATE_OPEN_KEEPALIVE2_ACK;
0361           } else if (tag == CEPH_MSGR_TAG_ACK) {
0362             state = STATE_OPEN_TAG_ACK;
0363           } else if (tag == CEPH_MSGR_TAG_MSG) {
0364             state = STATE_OPEN_MESSAGE_HEADER;
0365           } else if (tag == CEPH_MSGR_TAG_CLOSE) {
0366             state = STATE_OPEN_TAG_CLOSE;
0367           } else {
0368             ldout(async_msgr->cct, 0) << __func__ << " bad tag " << (int)tag << dendl;
0369             goto fail;
0370           }
0371
0372           break;
0373         }
```

## STATE_OPEN_MESSAGE_HEADER
- 根據 feature 的值，決定要走新版還是舊版的 header
- 讀取 header 大小的資料，然後將需要的資料都抓出來記錄下來。
- 驗證對方送來資料的 CRC 是否正確
- 將相關資料給 reset (這些結構都跟 header 有關)
    - data_buf
    - front
    - middle
    - data
- 記錄收到時間的時間戳
- 將狀態改變成 **STATE_OPEN_MESSAGE_THROTTLE_MESSAGE**

``` C++
0435       case STATE_OPEN_MESSAGE_HEADER:
0436         {
0437 #if defined(WITH_LTTNG) && defined(WITH_EVENTTRACE)
0438           ltt_recv_stamp = ceph_clock_now();
0439 #endif
0440           ldout(async_msgr->cct, 20) << __func__ << " begin MSG" << dendl;
0441           ceph_msg_header header;
0442           ceph_msg_header_old oldheader;
0443           __u32 header_crc = 0;
0444           unsigned len;
0445           if (has_feature(CEPH_FEATURE_NOSRCADDR))
0446             len = sizeof(header);
0447           else
0448             len = sizeof(oldheader);
0449
0450           r = read_until(len, state_buffer);
0451           if (r < 0) {
0452             ldout(async_msgr->cct, 1) << __func__ << " read message header failed" << dendl;
0453             goto fail;
0454           } else if (r > 0) {
0455             break;
0456           }
0457
0458           ldout(async_msgr->cct, 20) << __func__ << " got MSG header" << dendl;
0459
0460           if (has_feature(CEPH_FEATURE_NOSRCADDR)) {
0461             header = *((ceph_msg_header*)state_buffer);
0462             if (msgr->crcflags & MSG_CRC_HEADER)
0463               header_crc = ceph_crc32c(0, (unsigned char *)&header,
0464                                        sizeof(header) - sizeof(header.crc));
0465           } else {
0466             oldheader = *((ceph_msg_header_old*)state_buffer);
0467             // this is fugly
0468             memcpy(&header, &oldheader, sizeof(header));
0469             header.src = oldheader.src.name;
0470             header.reserved = oldheader.reserved;
0471             if (msgr->crcflags & MSG_CRC_HEADER) {
0472               header.crc = oldheader.crc;
0473               header_crc = ceph_crc32c(0, (unsigned char *)&oldheader, sizeof(oldheader) - sizeof(oldheader.crc));
0474             }
0475           }
0476
0477           ldout(async_msgr->cct, 20) << __func__ << " got envelope type=" << header.type
0478                               << " src " << entity_name_t(header.src)
0479                               << " front=" << header.front_len
0480                               << " data=" << header.data_len
0481                               << " off " << header.data_off << dendl;
0482
0483           // verify header crc
0484           if (msgr->crcflags & MSG_CRC_HEADER && header_crc != header.crc) {
0485             ldout(async_msgr->cct,0) << __func__ << " got bad header crc "
0486                                      << header_crc << " != " << header.crc << dendl;
0487             goto fail;
0488           }
0489
0490           // Reset state
0491           data_buf.clear();
0492           front.clear();
0493           middle.clear();
0494           data.clear();
0495           recv_stamp = ceph_clock_now();
0496           current_header = header;
0497           state = STATE_OPEN_MESSAGE_THROTTLE_MESSAGE;
0498           break;
0499         }
```


## STATE_OPEN_MESSAGE_THROTTLE_MESSAGE
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
- 這個 function 檢查是否有 **throttle** 訊息的數量限制，若有限制且超過上限，則建立一個 time event，並儲存下來。
- 最後將狀態切換到 **STATE_OPEN_MESSAGE_THROTTLE_BYTES**

```C++
0501       case STATE_OPEN_MESSAGE_THROTTLE_MESSAGE:
0502         {
0503           if (policy.throttler_messages) {
0504             ldout(async_msgr->cct, 10) << __func__ << " wants " << 1 << " message from policy throttler "
0505                                        << policy.throttler_messages->get_current() << "/"
0506                                        << policy.throttler_messages->get_max() << dendl;
0507             if (!policy.throttler_messages->get_or_fail()) {
0508               ldout(async_msgr->cct, 10) << __func__ << " wants 1 message from policy throttle "
0509                                          << policy.throttler_messages->get_current() << "/"
0510                                          << policy.throttler_messages->get_max() << " failed, just wait." << dendl;
0511               // following thread pool deal with th full message queue isn't a
0512               // short time, so we can wait a ms.
0513               if (register_time_events.empty())
0514                 register_time_events.insert(center->create_time_event(1000, wakeup_handler));
0515               break;
0516             }
0517           }
0518
0519           state = STATE_OPEN_MESSAGE_THROTTLE_BYTES;
0520           break;
0521         }
```

## STATE_OPEN_MESSAGE_THROTTLE_BYTES;
- 從 **connection** 讀取資料，分別對應到 header 中的三個成員
    - front
    - middle
    - data
- 此 function 則是檢查 throttle 訊息的 bytes 數量，若數量超過上限，也是建議一個 time event並存下來，待之後處理
- 最後將狀態切換到 **STATE_OPEN_MESSAGE_THROTTLE_DISPATCH_QUEUE**

```C++
0523       case STATE_OPEN_MESSAGE_THROTTLE_BYTES:
0524         {
0525           cur_msg_size = current_header.front_len + current_header.middle_len + current_header.data_len;
0526           if (cur_msg_size) {
0527             if (policy.throttler_bytes) {
0528               ldout(async_msgr->cct, 10) << __func__ << " wants " << cur_msg_size << " bytes from policy throttler "
0529                                          << policy.throttler_bytes->get_current() << "/"
0530                                          << policy.throttler_bytes->get_max() << dendl;
0531               if (!policy.throttler_bytes->get_or_fail(cur_msg_size)) {
0532                 ldout(async_msgr->cct, 10) << __func__ << " wants " << cur_msg_size << " bytes from policy throttler "
0533                                            << policy.throttler_bytes->get_current() << "/"
0534                                            << policy.throttler_bytes->get_max() << " failed, just wait." << dendl;
0535                 // following thread pool deal with th full message queue isn't a
0536                 // short time, so we can wait a ms.
0537                 if (register_time_events.empty())
0538                   register_time_events.insert(center->create_time_event(1000, wakeup_handler));
0539                 break;
0540               }
0541             }
0542           }
0543
0544           state = STATE_OPEN_MESSAGE_THROTTLE_DISPATCH_QUEUE;
0545           break;
0546         }
```

## STATE_OPEN_MESSAGE_THROTTLE_DISPATCH_QUEUE
- 如果剛剛有在 **STATE_OPEN_MESSAGE_THROTTLE_BYTES** 讀取到 front/middle/data 的資料的話，則這邊要確認 disaptch 本身的 throttle 有沒有超過，若超過也是送一個 time event 待稍後再來重新試試看
- 紀錄 throttle 的時間戳
- 狀態切換成 **STATE_OPEN_MESSAGE_READ_FRONT**

``` c++
0548       case STATE_OPEN_MESSAGE_THROTTLE_DISPATCH_QUEUE:
0549         {
0550           if (cur_msg_size) {
0551             if (!dispatch_queue->dispatch_throttler.get_or_fail(cur_msg_size)) {
0552               ldout(async_msgr->cct, 10) << __func__ << " wants " << cur_msg_size << " bytes from dispatch throttle "
0553                                          << dispatch_queue->dispatch_throttler.get_current() << "/"
0554                                          << dispatch_queue->dispatch_throttler.get_max() << " failed, just wait." << dendl;
0555               // following thread pool deal with th full message queue isn't a
0556               // short time, so we can wait a ms.
0557               if (register_time_events.empty())
0558                 register_time_events.insert(center->create_time_event(1000, wakeup_handler));
0559               break;
0560             }
0561           }
0562
0563           throttle_stamp = ceph_clock_now();
0564           state = STATE_OPEN_MESSAGE_READ_FRONT;
0565           break;
0566         }
```

## STATE_OPEN_MESSAGE_READ_FRONT
- 接下來就是開始讀取真正的封包資料了，主要是分成三個部分
    - front
    - middle
    - data
- 讀取前段資料，將內容先放到本身的 front 變數中
- 將狀態切成 **STATE_OPEN_MESSAGE_READ_MIDDLE**

```C++
0568       case STATE_OPEN_MESSAGE_READ_FRONT:
0569         {
0570           // read front
0571           unsigned front_len = current_header.front_len;
0572           if (front_len) {
0573             if (!front.length())
0574               front.push_back(buffer::create(front_len));
0575
0576             r = read_until(front_len, front.c_str());
0577             if (r < 0) {
0578               ldout(async_msgr->cct, 1) << __func__ << " read message front failed" << dendl;
0579               goto fail;
0580             } else if (r > 0) {
0581               break;
0582             }
0583
0584             ldout(async_msgr->cct, 20) << __func__ << " got front " << front.length() << dendl;
0585           }
0586           state = STATE_OPEN_MESSAGE_READ_MIDDLE;
0587         }
```

## STATE_OPEN_MESSAGE_READ_MIDDLE
- 讀取中段資料，將內容先放到本身的 middle 變數中
- 將狀態切成 **STATE_OPEN_MESSAGE_READ_DATA_PREPARE**

```C++
0589       case STATE_OPEN_MESSAGE_READ_MIDDLE:
0590         {
0591           // read middle
0592           unsigned middle_len = current_header.middle_len;
0593           if (middle_len) {
0594             if (!middle.length())
0595               middle.push_back(buffer::create(middle_len));
0596
0597             r = read_until(middle_len, middle.c_str());
0598             if (r < 0) {
0599               ldout(async_msgr->cct, 1) << __func__ << " read message middle failed" << dendl;
0600               goto fail;
0601             } else if (r > 0) {
0602               break;
0603             }
0604             ldout(async_msgr->cct, 20) << __func__ << " got middle " << middle.length() << dendl;
0605           }
0606
0607           state = STATE_OPEN_MESSAGE_READ_DATA_PREPARE;
0608         }
```

## STATE_OPEN_MESSAGE_READ_DATA_PREPARE
- 準備好 buffer 供之後讀取 data 用，其中 data 部分除了長度外，還有 **offset** 也要處理
- 這邊還不會讀取資料，只是會根據讀出來的空間長度預先配置一個空間供之後讀取資料使用
- 將狀態切成 STATE_OPEN_MESSAGE_READ_DATA

```C++
0610       case STATE_OPEN_MESSAGE_READ_DATA_PREPARE:
0611         {
0612           // read data
0613           unsigned data_len = le32_to_cpu(current_header.data_len);
0614           unsigned data_off = le32_to_cpu(current_header.data_off);
0615           if (data_len) {
0616             // get a buffer
0617             map<ceph_tid_t,pair<bufferlist,int> >::iterator p = rx_buffers.find(current_header.tid);
0618             if (p != rx_buffers.end()) {
0619               ldout(async_msgr->cct,10) << __func__ << " seleting rx buffer v " << p->second.second
0620                                   << " at offset " << data_off
0621                                   << " len " << p->second.first.length() << dendl;
0622               data_buf = p->second.first;
0623               // make sure it's big enough
0624               if (data_buf.length() < data_len)
0625                 data_buf.push_back(buffer::create(data_len - data_buf.length()));
0626               data_blp = data_buf.begin();
0627             } else {
0628               ldout(async_msgr->cct,20) << __func__ << " allocating new rx buffer at offset " << data_off << dendl;
0629               alloc_aligned_buffer(data_buf, data_len, data_off);
0630               data_blp = data_buf.begin();
0631             }
0632           }
0633
0634           msg_left = data_len;
0635           state = STATE_OPEN_MESSAGE_READ_DATA;
0636         }
```

## STATE_OPEN_MESSAGE_READ_DATA
- 透過一個迴圈嘗試將資料讀取出來並且放到之前所預先配置的空間 **DATA**
- 最後狀態切換到 **STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH**

```c++
0638       case STATE_OPEN_MESSAGE_READ_DATA:
0639         {
0640           while (msg_left > 0) {
0641             bufferptr bp = data_blp.get_current_ptr();
0642             unsigned read = MIN(bp.length(), msg_left);
0643             r = read_until(read, bp.c_str());
0644             if (r < 0) {
0645               ldout(async_msgr->cct, 1) << __func__ << " read data error " << dendl;
0646               goto fail;
0647             } else if (r > 0) {
0648               break;
0649             }
0650
0651             data_blp.advance(read);
0652             data.append(bp, 0, read);
0653             msg_left -= read;
0654           }
0655
0656           if (msg_left > 0)
0657             break;
0658
0659           state = STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH;
0660         }
0661
```

## STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH
- 這個 function 比較長，不過可以說是最後一步驟了。
- 跟 header 一樣，根據 **feature** 決定使用新舊版本的 **footer** 格式
- 讀取 footer 的資料，如各區段的CRC等
- 透過 decode_message 此 function，將收集到的 (front, middle, data, footer.etc) 組合成一個完整的 **message** 格式的封包
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
- 到這邊就結束了，接下來 **AsyncManager** 去定期去檢查 **dispatch_queue**，當有封包進來後，就會將該封包送給所有註冊的 **Dispatcher** 去處理。


```c++
0662       case STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH:
0663         {
0664           ceph_msg_footer footer;
0665           ceph_msg_footer_old old_footer;
0666           unsigned len;
0667           // footer
0668           if (has_feature(CEPH_FEATURE_MSG_AUTH))
0669             len = sizeof(footer);
0670           else
0671             len = sizeof(old_footer);
0672
0673           r = read_until(len, state_buffer);
0674           if (r < 0) {
0675             ldout(async_msgr->cct, 1) << __func__ << " read footer data error " << dendl;
0676             goto fail;
0677           } else if (r > 0) {
0678             break;
0679           }
0680
0681           if (has_feature(CEPH_FEATURE_MSG_AUTH)) {
0682             footer = *((ceph_msg_footer*)state_buffer);
0683           } else {
0684             old_footer = *((ceph_msg_footer_old*)state_buffer);
0685             footer.front_crc = old_footer.front_crc;
0686             footer.middle_crc = old_footer.middle_crc;
0687             footer.data_crc = old_footer.data_crc;
0688             footer.sig = 0;
0689             footer.flags = old_footer.flags;
0690           }
0691           int aborted = (footer.flags & CEPH_MSG_FOOTER_COMPLETE) == 0;
0692           ldout(async_msgr->cct, 10) << __func__ << " aborted = " << aborted << dendl;
0693           if (aborted) {
0694             ldout(async_msgr->cct, 0) << __func__ << " got " << front.length() << " + " << middle.length() << " + " << data.length()
0695                                 << " byte message.. ABORTED" << dendl;
0696             goto fail;
0697           }
0698
0699           ldout(async_msgr->cct, 20) << __func__ << " got " << front.length() << " + " << middle.length()
0700                               << " + " << data.length() << " byte message" << dendl;
0701           Message *message = decode_message(async_msgr->cct, async_msgr->crcflags, current_header, footer,
0702                                             front, middle, data, this);
0703           if (!message) {
0704             ldout(async_msgr->cct, 1) << __func__ << " decode message failed " << dendl;
0705             goto fail;
0706           }
0707
0708           //
0709           //  Check the signature if one should be present.  A zero return indicates success. PLR
0710           //
0711
0712           if (session_security.get() == NULL) {
0713             ldout(async_msgr->cct, 10) << __func__ << " no session security set" << dendl;
0714           } else {
0715             if (session_security->check_message_signature(message)) {
0716               ldout(async_msgr->cct, 0) << __func__ << " Signature check failed" << dendl;
0717               message->put();
0718               goto fail;
0719             }
0720           }
0721           message->set_byte_throttler(policy.throttler_bytes);
0722           message->set_message_throttler(policy.throttler_messages);
0723
0724           // store reservation size in message, so we don't get confused
0725           // by messages entering the dispatch queue through other paths.
0726           message->set_dispatch_throttle_size(cur_msg_size);
0727
0728           message->set_recv_stamp(recv_stamp);
0729           message->set_throttle_stamp(throttle_stamp);
0730           message->set_recv_complete_stamp(ceph_clock_now());
0731
0732           // check received seq#.  if it is old, drop the message.
0733           // note that incoming messages may skip ahead.  this is convenient for the client
0734           // side queueing because messages can't be renumbered, but the (kernel) client will
0735           // occasionally pull a message out of the sent queue to send elsewhere.  in that case
0736           // it doesn't matter if we "got" it or not.
0737           uint64_t cur_seq = in_seq.read();
0738           if (message->get_seq() <= cur_seq) {
0739             ldout(async_msgr->cct,0) << __func__ << " got old message "
0740                     << message->get_seq() << " <= " << cur_seq << " " << message << " " << *message
0741                     << ", discarding" << dendl;
0742             message->put();
0743             if (has_feature(CEPH_FEATURE_RECONNECT_SEQ) && async_msgr->cct->_conf->ms_die_on_old_message)
0744               assert(0 == "old msgs despite reconnect_seq feature");
0745             break;
0746           }
0747           if (message->get_seq() > cur_seq + 1) {
0748             ldout(async_msgr->cct, 0) << __func__ << " missed message?  skipped from seq "
0749                                       << cur_seq << " to " << message->get_seq() << dendl;
0750             if (async_msgr->cct->_conf->ms_die_on_skipped_message)
0751               assert(0 == "skipped incoming seq");
0752           }
0753
0754           message->set_connection(this);
0755
0756 #if defined(WITH_LTTNG) && defined(WITH_EVENTTRACE)
0757           if (message->get_type() == CEPH_MSG_OSD_OP || message->get_type() == CEPH_MSG_OSD_OPREPLY) {
0758             utime_t ltt_processed_stamp = ceph_clock_now();
0759             double usecs_elapsed = (ltt_processed_stamp.to_nsec()-ltt_recv_stamp.to_nsec())/1000;
0760             ostringstream buf;
0761             if (message->get_type() == CEPH_MSG_OSD_OP)
0762               OID_ELAPSED_WITH_MSG(message, usecs_elapsed, "TIME_TO_DECODE_OSD_OP", false);
0763             else
0764               OID_ELAPSED_WITH_MSG(message, usecs_elapsed, "TIME_TO_DECODE_OSD_OPREPLY", false);
0765           }
0766 #endif
0767
0768           // note last received message.
0769           in_seq.set(message->get_seq());
0770           ldout(async_msgr->cct, 5) << " rx " << message->get_source() << " seq "
0771                                     << message->get_seq() << " " << message
0772                                     << " " << *message << dendl;
0773
0774           if (!policy.lossy) {
0775             ack_left.inc();
0776             need_dispatch_writer = true;
0777           }
0778           state = STATE_OPEN;
0779
0780           logger->inc(l_msgr_recv_messages);
0781           logger->inc(l_msgr_recv_bytes, cur_msg_size + sizeof(ceph_msg_header) + sizeof(ceph_msg_footer));
0782
0783           async_msgr->ms_fast_preprocess(message);
0784           if (delay_state) {
0785             utime_t release = message->get_recv_stamp();
0786             double delay_period = 0;
0787             if (rand() % 10000 < async_msgr->cct->_conf->ms_inject_delay_probability * 10000.0) {
0788               delay_period = async_msgr->cct->_conf->ms_inject_delay_max * (double)(rand() % 10000) / 10000.0;
0789               release += delay_period;
0790               ldout(async_msgr->cct, 1) << "queue_received will delay until " << release << " on "
0791                                         << message << " " << *message << dendl;
0792             }
0793             delay_state->queue(delay_period, release, message);
0794           } else if (async_msgr->ms_can_fast_dispatch(message)) {
0795             lock.unlock();
0796             dispatch_queue->fast_dispatch(message);
0797             lock.lock();
0798           } else {
0799             dispatch_queue->enqueue(message, message->get_priority(), conn_id);
0800           }
0801
0802           break;
0803         }
0804
```

上述已經大概跑完了整個 **Accept** 的流程，當然此流程中是確保沒有任何錯誤，一切都是順利往下進行的。
接下來來探討若 **Client** 想要連線，則會怎麼處理。


# Connect
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


# _connect
- 將狀態設定為 **STATE_CONNECTING**，接下來就將 **read_handler** 送給 **Event Engine**去處理，由於是透過 **external event**，所以則會馬上執行 **read_handler**，也就是 **process**。
```c++
1856 void AsyncConnection::_connect()
1857 {
1858   ldout(async_msgr->cct, 10) << __func__ << " csq=" << connect_seq << dendl;
1859
1860   state = STATE_CONNECTING;
1861   // rescheduler connection in order to avoid lock dep
1862   // may called by external thread(send_message)
1863   center->dispatch_event_external(read_handler);
1864 }
```

## STATE_CONNECTING
- 檢查 cs 此變數，如果之前有連線過，則關閉先前的連線
    - 同時也先刪除之前的 event
- 呼叫 worker 跟對方的 socket 去連線
- 創造一個 read_handler 的 event，來處理接下來收到封包的行為
    - read_handler 就是 process
- 狀態切換成 STATE_CONNECTING_RE

```c++
0870     case STATE_CONNECTING:
0871       {
0872         assert(!policy.server);
0873
0874         // reset connect state variables
0875         got_bad_auth = false;
0876         delete authorizer;
0877         authorizer = NULL;
0878         authorizer_buf.clear();
0879         memset(&connect_msg, 0, sizeof(connect_msg));
0880         memset(&connect_reply, 0, sizeof(connect_reply));
0881
0882         global_seq = async_msgr->get_global_seq();
0883         // close old socket.  this is safe because we stopped the reader thread above.
0884         if (cs) {
0885           center->delete_file_event(cs.fd(), EVENT_READABLE|EVENT_WRITABLE);
0886           cs.close();
0887         }
0888
0889         SocketOptions opts;
0890         opts.priority = async_msgr->get_socket_priority();
0891         opts.connect_bind_addr = msgr->get_myaddr();
0892         r = worker->connect(get_peer_addr(), opts, &cs);
0893         if (r < 0)
0894           goto fail;
0895
0896         center->create_file_event(cs.fd(), EVENT_READABLE, read_handler);
0897         state = STATE_CONNECTING_RE;
0898         break;
0899       }

```

## STATE_CONNECTING_RE
- 檢查當前 connectionSocket 連線狀態，本身若發現沒有連線則會自己重新連線
    - r < 0, 連線依然失敗，則判定有問題， goto 離開
    - r == 1, 成功，不做事情
    - r == 0, 重連過程中有出現錯誤，可能是 EINPROGRESS  或是 EALREADY。
- 嘗試送出 CEPH_BANNER
- 若成功，狀態切換成 **STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY**
- 若失敗，狀態切換成 **STATE_WAIT_SEND**，待之後重送後再處理。

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
0901     case STATE_CONNECTING_RE:
0902       {
0903         r = cs.is_connected();
0904         if (r < 0) {
0905           ldout(async_msgr->cct, 1) << __func__ << " reconnect failed " << dendl;
0906           if (r == -ECONNREFUSED) {
0907             ldout(async_msgr->cct, 2) << __func__ << " connection refused!" << dendl;
0908             dispatch_queue->queue_refused(this);
0909           }
0910           goto fail;
0911         } else if (r == 0) {
0912           ldout(async_msgr->cct, 10) << __func__ << " nonblock connect inprogress" << dendl;
0913           if (async_msgr->get_stack()->nonblock_connect_need_writable_event())
0914             center->create_file_event(cs.fd(), EVENT_WRITABLE, read_handler);
0915           break;
0916         }
0917
0918         center->delete_file_event(cs.fd(), EVENT_WRITABLE);
0919         ldout(async_msgr->cct, 10) << __func__ << " connect successfully, ready to send banner" << dendl;
0920
0921         bufferlist bl;
0922         bl.append(CEPH_BANNER, strlen(CEPH_BANNER));
0923         r = try_send(bl);
0924         if (r == 0) {
0925           state = STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY;
0926           ldout(async_msgr->cct, 10) << __func__ << " connect write banner done: "
0927                                      << get_peer_addr() << dendl;
0928         } else if (r > 0) {
0929           state = STATE_WAIT_SEND;
0930           state_after_send = STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY;
0931           ldout(async_msgr->cct, 10) << __func__ << " connect wait for write banner: "
0932                                << get_peer_addr() << dendl;
0933         } else {
0934           goto fail;
0935         }
0936
0937         break;
0938       }
0223 }
```


## STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY
- 讀取 SERVER 端送來的資訊
	-  CEPH_BANNER
	-  Address (Server本身，以及Server看到的 Client)，所以有兩份。
- 比較兩邊的 CEPH_BANNER
- 比較 peer addr (server address)
    - 我自己 socket 看到的
    - 對方送過來的
- 將自己的 address 送給 server
- 若成功，將狀態切換成 **STATE_CONNECTING_SEND_CONNECT_MSG**
- 若失敗，將狀態切換成 **STATE_WAIT_SEND**，之後再處理。

```c++
0940     case STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY:
0941       {
0942         entity_addr_t paddr, peer_addr_for_me;
0943         bufferlist myaddrbl;
0944         unsigned banner_len = strlen(CEPH_BANNER);
0945         unsigned need_len = banner_len + sizeof(ceph_entity_addr)*2;
0946         r = read_until(need_len, state_buffer);
0947         if (r < 0) {
0948           ldout(async_msgr->cct, 1) << __func__ << " read banner and identify addresses failed" << dendl;
0949           goto fail;
0950         } else if (r > 0) {
0951           break;
0952         }
0953
0954         if (memcmp(state_buffer, CEPH_BANNER, banner_len)) {
0955           ldout(async_msgr->cct, 0) << __func__ << " connect protocol error (bad banner) on peer "
0956                                     << get_peer_addr() << dendl;
0957           goto fail;
0958         }
0959
0960         bufferlist bl;
0961         bl.append(state_buffer+banner_len, sizeof(ceph_entity_addr)*2);
0962         bufferlist::iterator p = bl.begin();
0963         try {
0964           ::decode(paddr, p);
0965           ::decode(peer_addr_for_me, p);
0966         } catch (const buffer::error& e) {
0967           lderr(async_msgr->cct) << __func__ <<  " decode peer addr failed " << dendl;
0968           goto fail;
0969         }
0970         ldout(async_msgr->cct, 20) << __func__ <<  " connect read peer addr "
0971                              << paddr << " on socket " << cs.fd() << dendl;
0972         if (peer_addr != paddr) {
0973           if (paddr.is_blank_ip() && peer_addr.get_port() == paddr.get_port() &&
0974               peer_addr.get_nonce() == paddr.get_nonce()) {
0975             ldout(async_msgr->cct, 0) << __func__ <<  " connect claims to be " << paddr
0976                                 << " not " << peer_addr
0977                                 << " - presumably this is the same node!" << dendl;
0978           } else {
0979             ldout(async_msgr->cct, 0) << __func__ << " connect claims to be "
0980                                 << paddr << " not " << peer_addr << " - wrong node!" << dendl;
0981             goto fail;
0982           }
0983         }
0984
0985         ldout(async_msgr->cct, 20) << __func__ << " connect peer addr for me is " << peer_addr_for_me << dendl;
0986         lock.unlock();
0987         async_msgr->learned_addr(peer_addr_for_me);
0988         if (async_msgr->cct->_conf->ms_inject_internal_delays) {
0989           if (rand() % async_msgr->cct->_conf->ms_inject_socket_failures == 0) {
0990             ldout(msgr->cct, 10) << __func__ << " sleep for "
0991                                  << async_msgr->cct->_conf->ms_inject_internal_delays << dendl;
0992             utime_t t;
0993             t.set_from_double(async_msgr->cct->_conf->ms_inject_internal_delays);
0994             t.sleep();
0995           }
0996         }
0997
0998         lock.lock();
0999         if (state != STATE_CONNECTING_WAIT_BANNER_AND_IDENTIFY) {
1000           ldout(async_msgr->cct, 1) << __func__ << " state changed while learned_addr, mark_down or "
1001                                     << " replacing must be happened just now" << dendl;
1002           return 0;
1003         }
1004
1005         ::encode(async_msgr->get_myaddr(), myaddrbl, 0); // legacy
1006         r = try_send(myaddrbl);
1007         if (r == 0) {
1008           state = STATE_CONNECTING_SEND_CONNECT_MSG;
1009           ldout(async_msgr->cct, 10) << __func__ << " connect sent my addr "
1010               << async_msgr->get_myaddr() << dendl;
1011         } else if (r > 0) {
1012           state = STATE_WAIT_SEND;
1013           state_after_send = STATE_CONNECTING_SEND_CONNECT_MSG;
1014           ldout(async_msgr->cct, 10) << __func__ << " connect send my addr done: "
1015               << async_msgr->get_myaddr() << dendl;
1016         } else {
1017           ldout(async_msgr->cct, 2) << __func__ << " connect couldn't write my addr, "
1018               << cpp_strerror(r) << dendl;
1019           goto fail;
1020         }
1021
1022         break;
1023       }
```


## STATE_CONNECTING_SEND_CONNECT_MSG
- 設定 **connect_msg** 的資訊
	- 如同之前所述，包含 **featrue**, **type**等。
- 將該 **connect_msg** 的資訊封裝起來到 **bl**變數中，透過 **try_send**送出
- 若成功則將狀態切換到 **STATE_CONNECTING_WAIT_CONNECT_REPLY**

```c++
1025     case STATE_CONNECTING_SEND_CONNECT_MSG:
1026       {
1027         if (!got_bad_auth) {
1028           delete authorizer;
1029           authorizer = async_msgr->get_authorizer(peer_type, false);
1030         }
1031         bufferlist bl;
1032
1033         connect_msg.features = policy.features_supported;
1034         connect_msg.host_type = async_msgr->get_myinst().name.type();
1035         connect_msg.global_seq = global_seq;
1036         connect_msg.connect_seq = connect_seq;
1037         connect_msg.protocol_version = async_msgr->get_proto_version(peer_type, true);
1038         connect_msg.authorizer_protocol = authorizer ? authorizer->protocol : 0;
1039         connect_msg.authorizer_len = authorizer ? authorizer->bl.length() : 0;
1040         if (authorizer)
1041           ldout(async_msgr->cct, 10) << __func__ <<  " connect_msg.authorizer_len="
1042                                      << connect_msg.authorizer_len << " protocol="
1043                                      << connect_msg.authorizer_protocol << dendl;
1044         connect_msg.flags = 0;
1045         if (policy.lossy)
1046           connect_msg.flags |= CEPH_MSG_CONNECT_LOSSY;  // this is fyi, actually, server decides!
1047         bl.append((char*)&connect_msg, sizeof(connect_msg));
1048         if (authorizer) {
1049           bl.append(authorizer->bl.c_str(), authorizer->bl.length());
1050         }
1051         ldout(async_msgr->cct, 10) << __func__ << " connect sending gseq=" << global_seq << " cseq="
1052             << connect_seq << " proto=" << connect_msg.protocol_version << dendl;
1053
1054         r = try_send(bl);
1055         if (r == 0) {
1056           state = STATE_CONNECTING_WAIT_CONNECT_REPLY;
1057           ldout(async_msgr->cct,20) << __func__ << " connect wrote (self +) cseq, waiting for reply" << dendl;
1058         } else if (r > 0) {
1059           state = STATE_WAIT_SEND;
1060           state_after_send = STATE_CONNECTING_WAIT_CONNECT_REPLY;
1061           ldout(async_msgr->cct, 10) << __func__ << " continue send reply " << dendl;
1062         } else {
1063           ldout(async_msgr->cct, 2) << __func__ << " connect couldn't send reply "
1064               << cpp_strerror(r) << dendl;
1065           goto fail;
1066         }
1067
1068         break;
1069       }
```

## STATE_CONNECTING_WAIT_CONNECT_REPLY
- 從 **Server** 端讀取資料，將資料放到 **state_buffer**，此資料的格式為 **ceph_msg_connect_reply**
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
- 將此資訊記錄下來後，狀態切換成 **STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH**

```c++
1071     case STATE_CONNECTING_WAIT_CONNECT_REPLY:
1072       {
1073         r = read_until(sizeof(connect_reply), state_buffer);
1074         if (r < 0) {
1075           ldout(async_msgr->cct, 1) << __func__ << " read connect reply failed" << dendl;
1076           goto fail;
1077         } else if (r > 0) {
1078           break;
1079         }
1080
1081         connect_reply = *((ceph_msg_connect_reply*)state_buffer);
1082
1083         ldout(async_msgr->cct, 20) << __func__ << " connect got reply tag " << (int)connect_reply.tag
1084                              << " connect_seq " << connect_reply.connect_seq << " global_seq "
1085                              << connect_reply.global_seq << " proto " << connect_reply.protocol_version
1086                              << " flags " << (int)connect_reply.flags << " features "
1087                              << connect_reply.features << dendl;
1088         state = STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH;
1089
1090         break;
1091       }
```


## STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH
- 若前述回傳的 **ceph_msg_connect_reply** 有 **authorizer** 的資訊，則進行額外處理。
- 最後呼叫 `handle_connect_reply` 進行處理
	- 理論上 **handle_connect_reply** 會改變當前狀態，最後變成 **STATE_CONNECTING_READY**
- 上述處理完畢後，狀態必須要改變，若沒有代表有問題，直接 **Assert**

```
1093     case STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH:
1094       {
1095         bufferlist authorizer_reply;
1096         if (connect_reply.authorizer_len) {
1097           ldout(async_msgr->cct, 10) << __func__ << " reply.authorizer_len=" << connect_reply.authorizer_len << dendl;
1098           assert(connect_reply.authorizer_len < 4096);
1099           r = read_until(connect_reply.authorizer_len, state_buffer);
1100           if (r < 0) {
1101             ldout(async_msgr->cct, 1) << __func__ << " read connect reply authorizer failed" << dendl;
1102             goto fail;
1103           } else if (r > 0) {
1104             break;
1105           }
1106
1107           authorizer_reply.append(state_buffer, connect_reply.authorizer_len);
1108           bufferlist::iterator iter = authorizer_reply.begin();
1109           if (authorizer && !authorizer->verify_reply(iter)) {
1110             ldout(async_msgr->cct, 0) << __func__ << " failed verifying authorize reply" << dendl;
1111             goto fail;
1112           }
1113         }
1114         r = handle_connect_reply(connect_msg, connect_reply);
1115         if (r < 0)
1116           goto fail;
1117
1118         // state must be changed!
1119         assert(state != STATE_CONNECTING_WAIT_CONNECT_REPLY_AUTH);
1120         break;
1121       }
```

## STATE_CONNECTING_READY
- 這邊有個有趣的註解 **// hooray!**，代表到這一步連線基本上已經完成了，剩下最後一步驟就結束了。
- 對 dispatch_queue 設定當前 connection
    - dispatch_queue 這邊有兩種類型，一種是存放 message，一種則是 Type + Connection，這邊屬於第二種
    - 在 dispatch_queue 的 loop 中，會針對這兩種去處理，若是 messag 的，則直接將此訊息丟給所有註冊的 dispatcher，反之則根據 type 執行不同的任務
    - 這邊放入的是 D_CONNECT 的 event，所以之後會執行 `ms_deliver_handle_connect` 這支 function。
    - 接者這支 function 則是會通知所有 dispathcer 目前有新的連線到來，呼叫對應的 `ms_handle_connect`來處理
- 呼叫 AsyncMessegner 內的 **ms_deliver_handle_fast_connect**
    - fast_connect 相對於 connect 是更早會處理的函式，底層可確保此 function 一定會在有任何 message 被處理前先呼叫。
- 如果當前 queue 內有訊息，這時候再發送一個外部的 write_handler把queue給清空。
    - 可能是由於先前的 try_send 沒有成功
- 到這邊後，連線就完成了，可以開始供應用層各種發送訊息了。

``` c++
1163     case STATE_CONNECTING_READY:
1164       {
1165         // hooray!
1166         peer_global_seq = connect_reply.global_seq;
```

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



上述在 **Accpeting** 或者是 **Connecting** 的過程中，會透過下列兩個方式做一些深層的處理，這些處理同時會改變當前狀態，這邊就簡單大致上看過而已。

handle_connect_reply
--------------------
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

# Summary
此 **AsyncConnection** 內容眾多，目前先主要觀察到整個建立連線的步驟，包含了 **Accept** 以及 **connect** 。
有機會再來把 **read/write** 相關的介面也都看一次，到時候可以更瞭解整體收送的行為。
