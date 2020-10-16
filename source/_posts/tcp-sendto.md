---
layout: post
title: TCP使用sendto
date: '2013-03-31 00:13'
comments: true
tags:
  - System
  - C
  - Network
abbrlink: 9548
---


OS:Linux

以前在寫Socket Programming的時候，對於TCP跟UDP在使用上會有一些區別
TCP要先建立連線，接者透過該連線把資料送出去，而UDP因為沒有連線，每次送出資料時都要指定對方的位置
寫TCP的時候，我習慣使用write跟send兩個function 來傳送資料
寫UDP的時候，我習慣使用sendto來傳送資料


``` c
     ssize_t
     send(int s, const void *msg, size_t len, int flags);

     ssize_t
     sendto(int s, const void *msg, size_t len, int flags,
         const struct sockaddr *to, socklen_t tolen);
```

<!--more-->



但是近期聽學長說，看到有程式碼以sendto來傳送資料，這是我第一次聽到，好奇之下，便去查詢了一下
這邊以linux-3.5.4版本為例

當使用者呼叫send來傳送資料時，會先呼叫
``` c
SYSCALL_DEFINE4(send, int, fd, void __user *, buff, size_t, len,
        unsigned int, flags)
{
    return sys_sendto(fd, buff, len, flags, NULL, 0);
}
```
這邊可以看到，send做的事情非常簡單，就是在去呼叫sendto，然後後面兩個位置的部分就給他填為NULL，所以對TCP連線來說，使用sendto並且後兩個參數也給NULL，也一樣可以work。

那這邊就好奇了，既然同樣都是使用sendto來傳送資料，那TCP沒有給定位置是因為本身已經有連線了，
那到底在sendto中是如何辦到這件事情了，所以又繼續往下看

``` c
SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
        unsigned int, flags, struct sockaddr __user *, addr,
        int, addr_len)
{
    struct socket *sock;
    struct sockaddr_storage address;
    int err;
    struct msghdr msg;
    struct iovec iov;
    int fput_needed;

    if (len > INT_MAX)
        len = INT_MAX;
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;

    iov.iov_base = buff;
    iov.iov_len = len;
    msg.msg_name = NULL;
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    msg.msg_control = NULL;
    msg.msg_controllen = 0;
    msg.msg_namelen = 0;
    if (addr) {
        err = move_addr_to_kernel(addr, addr_len, &address);
        if (err < 0)
            goto out_put;
        msg.msg_name = (struct sockaddr *)&address;
        msg.msg_namelen = addr_len;
    }
    if (sock->file->f_flags & O_NONBLOCK)
        flags |= MSG_DONTWAIT;
    msg.msg_flags = flags;
    err = sock_sendmsg(sock, &msg, len);

out_put:
    fput_light(sock->file, fput_needed);
out:
    return err;
}

```
這邊可以看到會利用sock->file->f_flags & O_NONBLOCK來檢查是否是個nonblock的傳送。
回歸正題，先執行move_addr_to_kernel這個function,把對方位置給轉移到kernel space中，
如果是TCP連線的話，傳進去的參數就會是NULL跟0，而

``` c
int move_addr_to_kernel(void __user *uaddr, int ulen, struct sockaddr_storage *kaddr)
{
    if (ulen < 0 || ulen > sizeof(struct sockaddr_storage))
        return -EINVAL;
    if (ulen == 0)
        return 0;
    if (copy_from_user(kaddr, uaddr, ulen))
        return -EFAULT;
    return audit_sockaddr(ulen, kaddr);
}

```

可以看到，當傳入的ulen是0的時候，就會回傳0，因此這邊對於TCP就不會回傳錯誤。
這邊可以看到

	msg.msg_name = (struct sockaddr *)&address;
	msg.msg_namelen = addr_len;
這邊可以看到會把對方位置的相關資訊給存到msg中，估計是之後UDP會用到
接下來透過sock_sendmsg傳送資料
sock_sendmsg ->__sock_sendmsg->__sock_sendmsg_nosec

```c
static inline int __sock_sendmsg_nosec(struct kiocb *iocb, struct socket *sock,
                       struct msghdr *msg, size_t size)
{
    struct sock_iocb *si = kiocb_to_siocb(iocb);

    sock_update_classid(sock->sk);

    sock_update_netprioidx(sock->sk);

    si->sock = sock;
    si->scm = NULL;
    si->msg = msg;
    si->size = size;

    return sock->ops->sendmsg(iocb, sock, msg, size);
}


```
這邊可以看到 最後會透過sock->ops->sendmsg(iocb, sock, msg, size) 這行把資料送出去
根據socket的種類是TCP還是UDP，對應到不同的function pointer
分別是tcp_sendmsg,udp_sendmsg

而在udp_sendmsg中就會去使用到剛剛在sendto那邊設定的msg
節錄自udp_sendmsg
這邊可以明顯看到會把msg中關於對方位置的資訊給抓出來，然後設定到daddr以及dport

``` c
	if(msg->msg_name) {
        struct sockaddr_in *usin = (struct sockaddr_in *)msg->msg_name;
        if (msg->msg_namelen < sizeof(*usin))
            return -EINVAL;
        if (usin->sin_family != AF_INET) {
            if (usin->sin_family != AF_UNSPEC)
                return -EAFNOSUPPORT;
        }

		daddr = usin->sin_addr.s_addr;
		dport = usin->sin_port;
		if (dport == 0)
		    return -EINVAL;
    } else {
		if (sk->sk_state != TCP_ESTABLISHED)
		    return -EDESTADDRREQ;
		daddr = inet->inet_daddr;
		dport = inet->inet_dport;
		/* Open fast path for connected socket.
		   Route will not be used, if at least one option is set.
		 */
		connected = 1;
    }

```

此外，recv以及recvfrom也是一樣的組合，與sent和sendto的關係差不多

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/D7RZGWrNK

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