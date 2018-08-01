---
title: Ceph Network Architecture 研究(三)
tags:
  - Ceph
  - Network
  - SOS
  - ScaleOut
  - SDS
  - SourceCode
keywords: 'Ceph,Network,SourceCode,SDS,SOS'
abbrlink: 25950
date: 2017-05-25 10:15:24
---

# Introduction

**Async** 希望在與底層kernel socket進行I/O處理時是以 **Async** 的方式去運行，而不是像 **Simple** 一樣每條 connetion 都要開兩個 **threads** 來負責處理 read 跟 write。
而現在普遍人在撰寫 **Network Programming** 時，幾乎都會採用 event-driven 類型的方式來處理，如 select.

在 **Async** 中負責進行上述這類型 **event-driven I/O** 處理的就是 **Event** 物件
**Event** 是一個抽象的介面，提供API給上層的 **Worker** 使用，而各種不同的類型的 **I/O**  都必須要繼承**Event**物件，並且實作所有的API。
目前的Event物件中有提供下列實作
- DPDK
- EPOLL
- KQUEUE
- SELECT

<!--more-->

在程式碼內，習慣以**driver**的字眼來稱呼這些方法，如 **SelectDriver**。 
```c++
0110   if (t == "dpdk") {
0111 #ifdef HAVE_DPDK
0112     driver = new DPDKDriver(cct);
0113 #endif
0114   } else {
0115 #ifdef HAVE_EPOLL
0116   driver = new EpollDriver(cct);
0117 #else
0118 #ifdef HAVE_KQUEUE
0119   driver = new KqueueDriver(cct);
0120 #else
0121   driver = new SelectDriver(cct);
0122 #endif
0123 #endif
0124   }
```

# Architecture
整個 **Event** 由下列物件組成
- EventCallback
- EventCenter
- EventDriver


接下來分別介紹這三個物件在整個運作中扮演的角色。

## EventCallBack
- 當事件發生時，要怎處理的介面雛形，所有的 CallBack 都必須要繼承此介面同時實作 `do_request`， 此 function 會得到一個 input，告知哪一個 fd 有事件發生了，然後實作對應的事情即可
- 該 fd 到底是 read 還是 write 被呼叫，這個 **EventCallBack** 本身不處理，此邏輯交給 EventCenter 去處理，所以若你的 CallBack 要依據 read/write 有不同處理，請註冊兩種不同的 callBack 來使用
```c++
0054 class EventCallback {
0055 
0056  public:
0057   virtual void do_request(int fd_or_id) = 0;
0058   virtual ~EventCallback() {}       // we want a virtual destructor!!!
0059 };
```
    
## EventDriver
- 此物件是底層每個方法都需要實現的介面，基本上跟**Event**有關的操作都在這邊完成，譬如哪些**fd**要監聽，哪些不用，然後監聽結果為何等。

```c++
0068 /*
0069  * EventDriver is a wrap of event mechanisms depends on different OS.
0070  * For example, Linux will use epoll(2), BSD will use kqueue(2) and select will
0071  * be used for worst condition.
0072  */
0073 class EventDriver {
0074  public:
0075   virtual ~EventDriver() {}       // we want a virtual destructor!!!
0076   virtual int init(EventCenter *center, int nevent) = 0;
0077   virtual int add_event(int fd, int cur_mask, int mask) = 0;
0078   virtual int del_event(int fd, int cur_mask, int del_mask) = 0;
0079   virtual int event_wait(vector<FiredFileEvent> &fired_events, struct timeval *tp) = 0;
0080   virtual int resize_events(int newsize) = 0;
0081   virtual bool need_wakeup() { return true; }
0082 };
```
### init
- 進行一些初始化的動作，以 **select** 為範例，就會將 read/write 的 FD 都初始化
### add_event
- 加入一個新的 **fd** 到需要觀察的清單中，而 **mask** 則有三種類型，分別是初始化/可讀/可寫。

```c++
0048 #define EVENT_NONE 0
0049 #define EVENT_READABLE 1
0050 #define EVENT_WRITABLE 2
```

- 以 **select** 為範例，此 function 就是將該 fd 加入到 **fd set** 中，同時更新當前最大 fd 數值

### del_event
- 將一個目標 fd 從觀察清單中移除，譬如當連線斷線後，就不需要在監聽此事件。
- 以 **select** 為範例，就是使用 `FD_CLR` 將該 fd 清除

### event_wait
- 這邊是真正重要的地方，此 function 會必須要真正去得到有哪些**fd**有對應的 event 產生，然後將這些 event收集起來。
- 呼叫此 function 時，必須要傳入一個含有 **FiredFileEvent** 的 vector以及 timeout 的時間
- 若這次等待中，有任何 **event** 被觸發，就將該 fd 放到該 vector 中即可。
FiredFileEvent 包含兩個成員，一個是發生事件的 fd，以及其對應的 mask (read/write)

``` c++
0063 struct FiredFileEvent {
0064   int fd;
0065   int mask;
0066 };
```

### resize_event
- 目前只有 kqueue 在使用，當加入新的 fd 超過當前容納數量，會透過此 function 去更新
### need_wakup
- 預設是回傳 True，目前只有 DPDK 有重新定義，原因是因為 DPDK 跟其他三者 event-based 的模式不相同，主要是一直依賴 polling 的方式去問。
 -這邊比較有趣的是，DPDK 這種 polling-based 的也一併放到**event**的架構中實作，不過因為還沒有看完**DPDK**的實作，還沒看清他是如何轉換這兩邊概念的。


## EventCenter
此介面是用來給上層使用的，在這邊將**Event**分成三大類，分別是
- read/write call-back event
- time event (多少ms後執行）
- external event (馬上執行，可以想成時間為 0 的 time event)

``` c++
0087 class EventCenter {
0088  public:
......
0102   struct FileEvent {
0103     int mask;
0104     EventCallbackRef read_cb;
0105     EventCallbackRef write_cb;
0106     FileEvent(): mask(0), read_cb(NULL), write_cb(NULL) {}
0107   };
0108 
0109   struct TimeEvent {
0110     uint64_t id;
0111     EventCallbackRef time_cb;
0112 
0113     TimeEvent(): id(0), time_cb(NULL) {}
0114   };
0115 
.....
0178   int process_time_events();
.....
0201   // Used by internal thread
0202   int create_file_event(int fd, int mask, EventCallbackRef ctxt);
0203   uint64_t create_time_event(uint64_t milliseconds, EventCallbackRef ctxt);
0204   void delete_file_event(int fd, int mask);
0205   void delete_time_event(uint64_t id);
0206   int process_events(int timeout_microseconds);
0207   void wakeup();
```

- 此物件內部還有在包含一個 **Poller** 的物件，主要是給DPDK使用
- 此外，針對非DPDK需要 wakeup 類型的 driver，如 EPOLL/KQUEUE/SELECT，實作了一個 read/write 的通知事件，為了避免 **driver** 卡在 **wait** 事件中，會透過 `pipe` 於本地創立一個專用的 FD，針對其 read fd 創造一個簡單的 read handler，單純讀取而已。
- 之後就透過一個只會寫入特定字元的 write event 使得該 driver 能夠從 wait 事件中出來。


``` c++
0038 class C_handle_notify : public EventCallback {
0039   EventCenter *center;
0040   CephContext *cct;
0041 
0042  public:
0043   C_handle_notify(EventCenter *c, CephContext *cc): center(c), cct(cc) {}
0044   void do_request(int fd_or_id) override {
0045     char c[256];
0046     int r = 0;
0047     do {
0048       r = read(fd_or_id, c, sizeof(c));
0049       if (r < 0) {
0050         if (errno != EAGAIN)
0051           ldout(cct, 1) << __func__ << " read notify pipe failed: " << cpp_strerror(errno) << dendl;
0052       }
0053     } while (r > 0);
0054   }
0055 };
```
``` c++
0315 void EventCenter::wakeup()
0316 {
0317   // No need to wake up since we never sleep
0318   if (!pollers.empty() || !driver->need_wakeup())
0319     return ;
0320 
0321   ldout(cct, 2) << __func__ << dendl;
0322   char buf = 'c';
0323   // wake up "event_wait"
0324   int n = write(notify_send_fd, &buf, sizeof(buf));
0325   if (n < 0) {
0326     if (errno != EAGAIN) {
0327       ldout(cct, 1) << __func__ << " write notify pipe failed: " << cpp_strerror(errno) << dendl;
0328       ceph_abort();
0329     }
0330   }
0331 }
```

### init
此 function 會決定底層要跑哪種**driver**，主要會基於參數**type**以及當前系統的平台，Linux優先走**Epoll**,
FreeBSD則是**Kqueue**,兩種都不符合的話就走**select**。
接下來呼叫該 **driver**的 init。
如果該 dirver 需要 wakeup (目前是除了DPDK以外)
- 透過 pipe 創建一對 local fd並且設定為 non-blocking
- 之後的 read/write notifier 會透過這對 fd 來傳輸。
    
### destructor
- 執行所有的 external events  並且從 queue 中移除。
- 若之前有透過 pipe 創立 local fd，將其關閉
- 移除 driver 以及供 wakup 使用的 notify_handler

### set_owner
- 讓該 **eventCenter** 記住擁有的 **thread** 是誰
    - 此變數主要會給 `in_thread` 這隻 function 來比較當前呼叫 event 的人是否是其owner。
- 創立一個(或是回傳已經存在的)全域的 event center
    - 此 global_cenets 是 `AssociatedCenters` 的物件
    - 裡面會存放 id 與 thread 的 mapping 關係
    - 主要是搭配 `submit_to` 使用，可以讓任意的人使用 `submit_to` 搭配特定的 id 馬上塞入一個 event 到對應的 thread 中
``` c++
0095   struct AssociatedCenters {
0096     EventCenter *centers[MAX_EVENTCENTER];
0097     AssociatedCenters(CephContext *c) {
0098       memset(centers, 0, MAX_EVENTCENTER * sizeof(EventCenter*));
0099     }
0100   };
```
- 若當前 driver 需要 wakeup，則創立一個 read evnet handler
``` c++
0196     if (driver->need_wakeup()) {
0197       notify_handler = new C_handle_notify(this, cct);
0198       int r = create_file_event(notify_receive_fd, EVENT_READABLE, notify_handler);
0199       assert(r == 0);
0200     }
```

### create_file_event
- 必須是同個 thread 才能夠透過此 function 來加入 event。
- 若傳入的 fd 大小超過當前的 fd上限，則透過 driver->resize_events 來調整
- 透過 `_get_file_event(fd)` 取得當前 FD 對應的 FileEvent
    - 此物件主要記錄當前 FD 對應的 mask
``` c++
0063 struct FiredFileEvent {
0064   int fd;
0065   int mask;
0066 };
```
- 若該 fd 以前就有 event，且其 mask 跟這次要加入的 mask 相同，那代表沒有任何改變，沒有必要繼續往下執行，故直接跳掉
- 接下來透過 driver->add_event 去創立該 event
    - 也有可能是修改已經存在 event 的 mask (read/write)
- 最後透過 mask 的數值設定其 read_cb/write_cb 對應的 event handler
``` c++
0237   event->mask |= mask;
0238   if (mask & EVENT_READABLE) {
0239     event->read_cb = ctxt;
0240   }
0241   if (mask & EVENT_WRITABLE) {
0242     event->write_cb = ctxt;
0243   }
```

### delete_file_event
- 跟 `create_file_event` 類似
- 若該 fd 超過目前已知FD的上限，代表有問題，輸出 log 並離開
- 透過 `_get_file_event(fd)` 取得該 fd 對應的 FileEvent
- 若對應的 mask 是0，代表此 fd 還沒有設定過任何的 event handler，所以不需要刪除，可以直接離開
- 呼叫 `driver->del_event` 刪除該 evnet
- 移除對應的 call back，並且修改該 event 的 mask
``` c++
0269   if (mask & EVENT_READABLE && event->read_cb) {
0270     event->read_cb = nullptr;
0271   }
0272   if (mask & EVENT_WRITABLE && event->write_cb) {
0273     event->write_cb = nullptr;
0274   }
0275 
0276   event->mask = event->mask & (~mask);
```

### create_time_event
- 透過 `clock_type::now()` 加上傳入的 **microseconds** 計算出 **expire** 的時間點
    - 使用 clock_type::time_point 當作該 event 要 expire 的時間點
- 透過 multimap 記住每個 time_point 對應的 event Handler
    - `std::multimap<clock_type::time_point, TimeEvent> time_events`
- 最後用一個 map  記住當前 id　對應上述 multimap 紀錄
    - id 則是用一個 global 的 time_event_next 來記住
    - 
``` c++
0288   clock_type::time_point expire = clock_type::now() + std::chrono::microseconds(microseconds);
0289   event.id = id;
0290   event.time_cb = ctxt;
0291   std::multimap<clock_type::time_point, TimeEvent>::value_type s_val(expire, event);
0292   auto it = time_events.insert(std::move(s_val));
0293   event_map[id] = it;
```

### delete_time_event
- 這邊是根據 id 來刪除對應的 time event
``` c++
0305   auto it = event_map.find(id);
0306   if (it == event_map.end()) {
0307     ldout(cct, 10) << __func__ << " id=" << id << " not found" << dendl;
0308     return ;
0309   }
0310 
0311   time_events.erase(it->second);
0312   event_map.erase(it);
```

### wakeup
- 判斷需不需要 wakeup
    - 若走DPDK，則需要看 pollers 是否空的，若非空則往下走
    - 其餘總類都必須要 wakeup
- 藉由寫入之前的 `notify_send_fd` 來叫起整 `event_wait`
- DPDK 的部分必須要再研究，私以為DPDK不應該下來，因為其 `notify_send_fd` 應該不會被初始化。

### process_time_events
- 處理所有的 time events
- 對於所有的 time event，如果當前的時間超過該 time event 的 expire time
    - 將該 event 從結構中移除
    - 呼叫該 event 的 call back function
    - 這邊傳入的是 ID，跟 FD 無關
    - 
- 回傳這次總共處理了多少個 event


### process_events
- 該 function 會傳入一個變數`timeout_microseconds`，這是給 driver 用的 timeout
- 符合下列情況，將 timeout 設定為0，這會讓 driver 變成 non-blocking
    - 存在外部 event
    - 存在 poller
- 否則，計算一下 timeout 的時候，讓 driver 會 block 一段時間
- 透過 driver->evnet_wait 去詢問當前有哪些 event ready 了
```c++
0394   vector<FiredFileEvent> fired_events;
0395   numevents = driver->event_wait(fired_events, &tv);
0396   for (int j = 0; j < numevents; j++) {
        ....
0419   }
```
- 對於所有被觸發的 event, 呼叫對應的 read/write handler
    - 若某個 FD同時可以進行 read/write  且對應的 call back handler 是相同的，則執行完 read 後，就不執行 write了 (不確定原因)
    - "可能"是避免重複執行相同內容，因為 function 沒有辦法知道當前被叫起是 read or write event ready。
```c++
0405     if (event->mask & fired_events[j].mask & EVENT_READABLE) {
0406       rfired = 1;
0407       cb = event->read_cb;
0408       cb->do_request(fired_events[j].fd);
0409     }
0410 
0411     if (event->mask & fired_events[j].mask & EVENT_WRITABLE) {
0412       if (!rfired || event->read_cb != event->write_cb) {
0413         cb = event->write_cb;
0414         cb->do_request(fired_events[j].fd);
0415       }
0416     }
```
- 嘗試執行 time_events，並且繼續記錄總共處理的 event 數量
- 若當前 external queue 內有東西，則將全部都執行完畢
- 若到現在都還沒有執行任何 event 且也不是 blocking mode，則呼叫 pollers去 polling 對應的 event，並且記錄下來總數
- 最後回傳總數量
``` c++
0421   if (trigger_time)
0422     numevents += process_time_events();
0423 
0424   if (external_num_events.load()) {
0425     external_lock.lock();
0426     deque<EventCallbackRef> cur_process;
0427     cur_process.swap(external_events);
0428     external_num_events.store(0);
0429     external_lock.unlock();
0430     while (!cur_process.empty()) {
0431       EventCallbackRef e = cur_process.front();
0432       ldout(cct, 20) << __func__ << " do " << e << dendl;
0433       e->do_request(0);
0434       cur_process.pop_front();
0435       numevents++;
0436     }
0437   }
0438 
0439   if (!numevents && !blocking) {
0440     for (uint32_t i = 0; i < pollers.size(); i++)
0441       numevents += pollers[i]->poll();
0442   }
0443 
0444   return numevents;
```


### dispatch_event_external
- 將 event 存放到 `external_events` 內
- 看看 `external_num_events` 這個變數是不是0，若是0則代表可以 wake
    - external_num_events 是個 atomic 類型的變數
    - 用此變數來控管當前是否正在清除 external_queue 內的 event
- 若符合下列條件，則呼叫 `wake` 將 event 叫起來
    - caller 的 thread 跟真正擁有此 eventCenter 的 thread 是相同
    - 前述的 `external_num_events` 決定當前需要 
``` c++
0450   external_events.push_back(e);
0451   bool wake = !external_num_events.load();
0452   uint64_t num = ++external_num_events;
0453   external_lock.unlock();
0454   if (!in_thread() && wake)
0455     wakeup();
```


### submit_to
- 此 function 可以將特定的 function 傳入給特定的 eventCenter，走 external event 的方式去執行
- 這邊的 event 會被包裝成 **C_submit_event**
    - 這邊 **do_eqeust** 會透過 condition_variable 與 **wait**來溝通
``` c++
0217   class C_submit_event : public EventCallback {
0218     std::mutex lock;
0219     std::condition_variable cond;
0220     bool done = false;
0221     func f;
0222     bool nonwait;
0223    public:
0224     C_submit_event(func &&_f, bool nw)
0225       : f(std::move(_f)), nonwait(nw) {}
0226     void do_request(int id) override {
0227       f();
0228       lock.lock();
0229       cond.notify_all();
0230       done = true;
0231       bool del = nonwait;
0232       lock.unlock();
0233       if (del)
0234         delete this;
0235     }
0236     void wait() {
0237       assert(!nonwait);
0238       std::unique_lock<std::mutex> l(lock);
0239       while (!done)
0240         cond.wait(l);
0241     }
0242   };
```
- 若 caller 跟該 EventCenter 屬於同個 thread，就直接執行了，不再塞到 exterbnl_queue
- 接下來根據變數中的 **no_wait**來決定要不要等待該 function 執行完畢
- 假如是 no_wait，則丟到 external_queue 後就直接離開
- 假如是 wait,則丟到 external_queue 後，**馬上呼叫 wait()**，等待 **do_request()**執行完畢後，會使得 **wait** 結束，然後離開此 function。


# Summary
整個 Event 系列的檔案其實不會太困難，除了 **DPDK** 本身還有額外的實作外，其餘 **POSIX** 系列的三種只要熟悉本身的用法，來看這些程式碼就不會覺得太陌生，所有的運作都可以想像的到，同時也在預料之中。
