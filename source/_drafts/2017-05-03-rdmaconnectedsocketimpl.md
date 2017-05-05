---
layout: post
title: 'RDMAConnectedSocketImpl'
date: 2017-05-03 02:49
comments: true
categories: 
---
RDMA - RDMAConnectedSocketImpl
==============================
- **RDMAConnectedSocketImpl** 繼承自 **ConnectedSocketImpl**，重新實做了下列 function
    - read
    - zero_copy_read
    - send
    - shutdown
    - close
    - fd
- 此物件主要實現了 RDMA 底層傳輸的 read/write
- RDMA 使用了 Async 的整體架構，但是因為整體架構本來是針對 event-driven 去設計的，所以 RDMA 這邊做了些小變化來迎合原先架構
	- 開始連線時偷偷用個 tcp 連線做些 control 訊息的交換(syn+ack)
	- 由於本身不走 event-driven，所以使用 eventfd 創了一個 local fd 給 event framework 使用以符合架構
	- 本身會定期 polling，當拿到資料後會寫資料到上述的 local fd裡面去驅動 event framework 有資料近來，接者 framework 就會去嘗試讀取資料，封裝最後送給每個 dispatcher 去處理。

#### constructor
- 從 **Infiniband** 物件中取得對應的 **ibdev**, **ibport**
- 創造 **queue_pair**
- 創造一個 **IBSYNMsg**，記住QP跟ibdev的資訊
```c++
0036   my_msg.qpn = qp->get_local_qp_number();
0037   my_msg.psn = qp->get_initial_psn();
0038   my_msg.lid = ibdev->get_lid();
0039   my_msg.peer_qpn = 0;
0040   my_msg.gid = ibdev->get_gid();
```
- 設定dispatcher的perf logger

#### deconstructor
- 呼叫 `cleanup`
- 從 worker 中移除當前這條 connection
- 移除相關的 fd
    - notify_fd
    - tcp_fd
- 待補

#### pass_wc
- wc 即是 **Work Completion**，根據這裡的[說明](http://www.rdmamojo.com/2013/02/15/ibv_poll_cq/)
>Work Completion means that the corresponding Work Request is ended and the buffer can be (re)used for read, write or free.
- 將 **ibv_wc** 的物件傳入並且記錄下來
- 若目前擁有的 wc 是空的，則直接接收參數
- 否則就把參數內的內容全部 insert 到 wc 這個變數中
``` c++
0072   if (wc.empty())
0073     wc = std::move(v);
0074   else
0075     wc.insert(wc.end(), v.begin(), v.end());
```

#### gwt_wc
- 取得當前完成的 Work Completion 資料
- 將目前擁有的 vector of wc 直接回傳
    - 透過 swap 的方式將內容直接丟回給 input
```c++
0079 void RDMAConnectedSocketImpl::get_wc(std::vector<ibv_wc> &w)
0080 {
0081   Mutex::Locker l(lock);
0082   if (wc.empty())
0083     return ;
0084   w.swap(wc);
0085 }
```

#### activate
- 取得當前 RDMA 使用的 **Device** 以及對應的 **ibport**
- 將前述透過 tcp control plane 取得的 peer qpn 給存放到 local 變數
```c++
0149   socket->remote_qpn = peer_msg.qpn;
```
- 狀態改變參考此[圖](http://www.rdmamojo.com/2012/05/05/qp-state-machine/)
- 將 QP 狀態轉換為 RTR
	- 將狀態設定為 RTR **(Ready To Receive)**
	- QP 的 MTU (目前只有1024)
- 將 QP 狀態轉換為 RTS
- 若當前不是 server
	- 拉起 flag 代表連線已經完畢
	- 呼叫 `socket->submit(false)`處理


#### try_connect
- client 使用，先透過 **net_handler** 去連接對方
- 設定 socket options
	- close_on_exec
	- nodelay
	- rcbuf_size
- 嘗試發送訊息過去
- 加入一個 readable event handler
	- handle_connection
```c++
0196   r = infiniband->send_msg(cct, tcp_fd, my_msg);
0197   if (r < 0)
0198     return r;
0199 
0200   worker->center.create_file_event(tcp_fd, EVENT_READABLE, con_handler);
```

#### handle_connection
- 這個 function 主要是 RDMA 的 tcp control message 使用的，作為一開始連線前的 syn/ack 溝通用
- 若是 server side，則當 tcp_fd(accept) 有任何 readable 的事件發生時，就會呼叫此 function 來處理。
- 若是 client side，則當 tcp_fd(connect) 有任何  readable 的事件發生時，就會呼叫此 function 來處理。
- 首先去接收訊息
	- 此 function 雖然在 infiniband 底下，但是實際上也是走 kernel 去接收 TCP 的訊息
``` c++
0206   int r = infiniband->recv_msg(cct, tcp_fd, peer_msg);
```
- 若是 client
	- 由於先前在 **try_connect** 時會送出 syn (peer_qpn == 0)，所以此時會期望可以從 server side 收到 syn + ack.
	- 如果這時侯 connected 還沒有 confirm，則透過 activate 去 confirm connected.
	- 發送 ack 訊息給 server
- 若是 server
	- 檢查收到的 peer_qpn
	- 若是0，則視為收到 client 送來的 syn，若還沒 activated 過，則回傳 ack 給 server 並且 activate
	- 若不是0，視為收到 client 送來的 ack，此時將 **connected** 拉起，並且清除該 fd read handler，最後呼叫 submit 以及 notify 處理後續。
- 上述的邏輯目前看起來有些地方需要釐清
	- 如果已經 activated 的話，就不會回傳 ack, 這樣 client 端那邊其實也走不到後面
	- client 端不會清除對應的 fd read handler
	- tcp_fd 沒有機會被 close (除了 destructor)

#### set_accept_fd
- 將先前透過 **RDMAServerSocketImpl** 創立的 TCP socket 給記錄下來
- 透過 eventCenter 創入一個 event
    - READABLE
    - callback handler 是 **handle_connection**
```c++
0624   tcp_fd = sd;
0625   is_server = true;
0626   worker->center.submit_to(worker->center.get_id(), [this]() {
0627                            worker->center.create_file_event(tcp_fd, EVENT_READABLE, con_handler);
0628                            }, true);
```

#### send
- 將要傳送的資料 **bl** 收集起來，放到 local 的 **pending_bl** 物件中
- 最後透過 `submit` 將資料真正發送出去

#### read
- 先將 notify_fd 內的東西讀出來，避免 read event 下次再度叫起
- 如果目前 buffers 內有東西的話，直接呼叫 `read_buffers` 將裡面的資料讀取放到 buf 上
	- buffers 會存放一堆 Chunk
- 呼叫 `get_wc`，看看當前是否有完成的任務
	- Work Completion
- 若沒有半個 wq 完成，就直接離開囉
	- 此時不代表沒有任何資料讀取成功，可能有資料會因為 `read_buffers` 而讀取到

#### read_buffers

#### post_read
- 無法理解用途XD
- 基本上功能跟 server 收到 connected 後的處理是一樣的，但是這邊不是很能理解為什麼這個時間點要這樣做



#### submit
- 嘗試送資料出去
	- 資料必須要事先存放於**pending_bl**此變數中
- lambada function
	- 先去跟 worker 要一些記憶體
	- 接下來嘗試將 data 都複製到 tx_buffers (tx_buffers 會在上述的要記憶體過程中變大)
	- 回傳總共複製了多少長度的單位
- 掃過全部要送出去的資料
	- 若該資料的 memory address 其實已經在 **cluster** 裡面，直接拿出對應的 chunk 放到 tx_buffers 即可
	- 否則就透過上述的 lambada function 將 **pending_bl** 內的變數一個一個複製到 tx_buffers 內的 chunks
- 若當前因為 memory 不夠導致沒有辦法送光全部的 data，則對 **pending_bl** 進行下列處理
	- 先透過 splice 將剩下的給放到區域變數 **swapped**
	- 再透過 swap 將  **swapped** 指派回 **pending_bl** 去
- 呼叫 **post_work_request** 將資料真正送出去
- 若還有資料要送，回傳 EAGAIN，否則0

#### post_work_request
- 將一系列的 **Chunk** 包裝到 work request **ibv_send_wr**中 
- 透過 **ibv_sge** 此結構記住要發送資料的記憶體位置
- 透過 **ibv_send_wr** 填寫相關資訊，然後把上述的 **ibv_sge** 串起來
- 將所有的 **ibv_send_wr** 串起來
- 呼叫 **ibv_post_send** 將此 work request 發送到 send queue pair 去。

#### fin
- 發送空的訊息到對面去

#### cleanup
- 將 tcp control plane 相關的事件都清除
	- 從 event center 內將該 handler 給移除
	- 將 handler 移除，設定為 nullptr

#### notify
- 寫一個資料到 **notify_fd** 內，讓該 fd 可以觸發 read event。
- 當 RDMA polling 到資料後，就會透過此方式讓 event framework 能夠知道有資料可以接收處理了


```c++
0501     Chunk *current_chunk = tx_buffers[chunk_idx];
0502     while (start != end) {
0503       const uintptr_t addr = reinterpret_cast<const uintptr_t>(start->c_str());
0504       unsigned copied = 0;
0505       while (copied < start->length()) {
0506         uint32_t r = current_chunk->write((char*)addr+copied, start->length() - copied);
0507         copied += r;
0508         total_copied += r;
0509         bytes -= r;
0510         if (current_chunk->full()){
0511           current_chunk = tx_buffers[++chunk_idx];
0512           if (chunk_idx == tx_buffers.size())
0513             return total_copied;
0514         }
0515       }
0516       ++start;
```

** polliing**
- First
	- read cq
	- read data from memory
	- check the event, (blocking function)
- Second
	- 透過 **ibv_post_recv**將 work request 放到 receive queue 內
	- 用 **ibv_poll_cq** 將已經處理完畢的 receive work request 收回來


RDNA Write, RDMA Read
- choose one to implement data transfer.

###### tags: `ceph` `rdma` `async` `msg`