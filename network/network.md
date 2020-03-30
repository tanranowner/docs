# 1. 协议模型

![img](E:\Repositories\docs\network\network.assets\v2-4fe0b5f06fc89af2f98ebd2690bc87ea_1440w.jpg)





## 1.1 TCP连接建立与关闭

tcp是面向连接的，可靠的传输协议。

### 1.1.1 三次握手

![img](E:\Repositories\docs\network\network.assets\20180717202520531.png)

第一次握手：建立连接时，客户端发送syn包（syn=x）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）。

第二次握手：服务器收到syn包，必须确认客户的SYN（ack=x+1），同时自己也发送一个SYN包（syn=y），即SYN+ACK包，此时服务器进入SYN_RECV状态；

第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=y+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手。

### 1.1.2 四次挥手

![img](E:\Repositories\docs\network\network.assets\20180717204202563.png)

1. 客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
2. 服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
3. 客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
4. 服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
5. 客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
6. 服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。

### 1.1.3 抓包

```shell
# 查看当前进程号
[root@zhaoyl ~]# echo $$
18653
# 创建tcp socket及其io（文件描述符88）
[root@zhaoyl ~]# exec 88<>/dev/tcp/www.baidu.com/80
# 查看io创建情况（io文件88已经创建成功）
[root@zhaoyl ~]# ll /proc/18653/fd
total 0
lrwx------ 1 root root 64 Mar 30 10:27 0 -> /dev/pts/2
lrwx------ 1 root root 64 Mar 30 10:27 1 -> /dev/pts/2
lrwx------ 1 root root 64 Mar 30 10:27 2 -> /dev/pts/2
lrwx------ 1 root root 64 Mar 30 10:27 255 -> /dev/pts/2
lrwx------ 1 root root 64 Mar 30 10:30 88 -> socket:[40843771]
# 通过进程号查询
[root@zhaoyl fd]# netstat -antp | grep 19125
tcp        1      0 172.17.233.46:57006     101.200.186.8:80        CLOSE_WAIT  19125/-bash
# 通过端口号查询
[root@zhaoyl fd]# netstat -antp | grep :80
tcp        0      0 0.0.0.0:8001            0.0.0.0:*               LISTEN      29070/java
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      30870/java
tcp        0      0 172.17.233.46:80        188.11.197.114:38630    ESTABLISHED 30870/java
tcp        0      0 172.17.233.46:50278     100.100.30.25:80        ESTABLISHED 12460/AliYunDun
tcp        0      0 172.17.233.46:57006     101.200.186.8:80        ESTABLISHED 19125/-bash

# 获取首页
[root@zhaoyl fd]# echo -e 'GET / HTTP/1.1 \n' >&8 || echo $?
[root@zhaoyl fd]# cat <&9
HTTP/1.1 400 Bad Request
Server: nginx/1.10.1
Date: Mon, 30 Mar 2020 03:24:46 GMT
Content-Type: text/html
Content-Length: 173
Connection: close

<html>
<head><title>400 Bad Request</title></head>
<body bgcolor="white">
<center><h1>400 Bad Request</h1></center>
<hr><center>nginx/1.10.1</center>
</body>
</html>
# 关闭tcp socket及其io（文件描述符88）
[root@zhaoyl ~]# echo 88>&-

```

三次握手和四次挥手抓包

```shell
# 终端1开启抓包
[root@zhaoyl ~]# tcpdump -nn -i eth0 host www.baidu.com
# 终端2开启连接
[root@zhaoyl ~]# curl www.baidu.com
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道……
# 终端1抓包情况
[root@zhaoyl ~]# tcpdump -nn -i eth0 host www.baidu.com
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes


21:03:12.902396 IP 172.17.233.46.49774 > 220.181.38.149.80: Flags [S], seq 1089446618, win 29200, options [mss 1460,sackOK,TS val 1687639620 ecr 0,nop,wscale 7], length 0
21:03:12.907284 IP 220.181.38.149.80 > 172.17.233.46.49774: Flags [S.], seq 2873299660, ack 1089446619, win 8192, options [mss 1452,sackOK,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,wscale 5], length 0
21:03:12.907307 IP 172.17.233.46.49774 > 220.181.38.149.80: Flags [.], ack 1, win 229, length 0
21:03:12.907359 IP 172.17.233.46.49774 > 220.181.38.149.80: Flags [P.], seq 1:78, ack 1, win 229, length 77: HTTP: GET / HTTP/1.1
21:03:12.912482 IP 220.181.38.149.80 > 172.17.233.46.49774: Flags [.], ack 78, win 916, length 0
21:03:12.913383 IP 220.181.38.149.80 > 172.17.233.46.49774: Flags [P.], seq 1:2782, ack 78, win 916, length 2781: HTTP: HTTP/1.1 200 OK
21:03:12.913398 IP 172.17.233.46.49774 > 220.181.38.149.80: Flags [.], ack 2782, win 272, length 0
21:03:12.913507 IP 172.17.233.46.49774 > 220.181.38.149.80: Flags [F.], seq 78, ack 2782, win 272, length 0
21:03:12.918619 IP 220.181.38.149.80 > 172.17.233.46.49774: Flags [.], ack 79, win 916, length 0
21:03:12.918684 IP 220.181.38.149.80 > 172.17.233.46.49774: Flags [F.], seq 2782, ack 79, win 916, length 0
21:03:12.918697 IP 172.17.233.46.49774 > 220.181.38.149.80: Flags [.], ack 2783, win 272, length 0
21:03:15.930388 IP 220.181.38.149.80 > 172.17.233.46.49774: Flags [R], seq 2873302443, win 0, length 0
```

握手和挥手数据：

![image-20200330210801110](E:\Repositories\docs\network\network.assets\image-20200330210801110.png)



flags字段及其含义：

```txt
SYN：请求建立连接，并在其序列号的字段进行序列号的初始值设定。建立连接，设置为1。
ACK：确认号是否有效，一般置为1。
PSH：提示接收端应用程序立即从TCP缓冲区把数据读走。
FIN：希望断开连接。
RST：对方要求重新建立连接，复位。
URG：紧急指针是否有效。为1，表示某一位需要被优先处理；
```

| socket状态   | 状态说明                                                     |
| ------------ | ------------------------------------------------------------ |
| CLOSED       | 没有使用这个套接字[netstat 无法显示closed状态]               |
| LISTEN       | 套接字正在监听连接[调用listen后]                             |
| ESTABLISHED  | 连接已建立                                                   |
| SYN_SENT     | 套接字正在试图主动建立连接[发送SYN后还没有收到ACK]           |
| SYN_RECEIVED | 正在处于连接的初始同步状态[收到对方的SYN，但还没收到自己发过去的SYN的ACK] |
| CLOSE_WAIT   | 远程套接字已经关闭：正在等待关闭这个套接字[被动关闭的一方收到FIN] |
| FIN_WAIT_1   | 套接字已关闭，正在关闭连接[发送FIN，没有收到ACK也没有收到FIN] |
| CLOSING      | 套接字已关闭，远程套接字正在关闭，暂时挂起关闭确认[在FIN_WAIT_1状态下收到被动方的FIN] |
| LAST_ACK     | 远程套接字已关闭，正在等待本地套接字的关闭确认[被动方在CLOSE_WAIT状态下发送FIN] |
| FIN_WAIT_2   | 套接字已关闭，正在等待远程套接字关闭[在FIN_WAIT_1状态下收到发过去FIN对应的ACK] |
| TIME_WAIT    | 这个套接字已经关闭，正在等待远程套接字的关闭传送[FIN、ACK、FIN、ACK都完毕，这是主动方的最后一个状态，在过了2MSL时间后变为CLOSED状态] |

# 2. FAQ

1. 为什么连接的时候是三次握手，关闭的时候却是四次握手？

答：因为当Server端收到Client端的SYN连接请求报文后，可以直接发送SYN+ACK报文。其中ACK报文是用来应答的，SYN报文是用来同步的。但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭SOCKET，所以只能先回复一个ACK报文，告诉Client端，"你发的FIN报文我收到了"。只有等到我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送。故需要四步握手。

2. 为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？

答：虽然按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假象网络是不可靠的，有可以最后一个ACK丢失。所以TIME_WAIT状态就是用来重发可能丢失的ACK报文。在Client发送出最后的ACK回复，但该ACK可能丢失。Server如果没有收到ACK，将不断重复发送FIN片段。所以Client不能立即关闭，它必须确认Server接收到了该ACK。Client会在发送出ACK之后进入到TIME_WAIT状态。Client会设置一个计时器，等待2MSL的时间。如果在该时间内再次收到FIN，那么Client会重发ACK并再次等待2MSL。所谓的2MSL是两倍的MSL(Maximum Segment Lifetime)。MSL指一个片段在网络中最大的存活时间，2MSL就是一个发送和一个回复所需的最大时间。如果直到2MSL，Client都没有再次收到FIN，那么Client推断ACK已经被成功接收，则结束TCP连接。

3. 为什么不能用两次握手进行连接？

答：3次握手完成两个重要的功能，既要双方做好发送数据的准备工作(双方都知道彼此已准备好)，也要允许双方就初始序列号进行协商，这个序列号在握手过程中被发送和确认。

       现在把三次握手改成仅需要两次握手，死锁是可能发生的。作为例子，考虑计算机S和C之间的通信，假定C给S发送一个连接请求分组，S收到了这个分组，并发 送了确认应答分组。按照两次握手的协定，S认为连接已经成功地建立了，可以开始发送数据分组。可是，C在S的应答分组在传输中被丢失的情况下，将不知道S 是否已准备好，不知道S建立什么样的序列号，C甚至怀疑S是否收到自己的连接请求分组。在这种情况下，C认为连接还未建立成功，将忽略S发来的任何数据分 组，只等待连接确认应答分组。而S在发出的分组超时后，重复发送同样的分组。这样就形成了死锁。

4. 如果已经建立了连接，但是客户端突然出现故障了怎么办？

TCP还设有一个保活计时器，显然，客户端如果出现故障，服务器不能一直等下去，白白浪费资源。服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是设置为2小时，若两小时还没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒钟发送一次。若一连发送10个探测报文仍然没反应，服务器就认为客户端出了故障，接着就关闭连接。