# 1. 协议模型

![img](E:\Repositories\docs\IO\IO模型.assets\v2-4fe0b5f06fc89af2f98ebd2690bc87ea_1440w.jpg)





## 1.1 TCP连接建立与关闭

tcp是面向连接的，可靠的传输协议。

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

三次握手和四次挥手抓包

```shell
# 终端1开启抓包
[root@zhaoyl ~]# tcpdump -nn -i eth0 host 220.181.38.150
# 终端2开启连接
[root@zhaoyl fd]# exec 8<>/dev/tcp/www.baidu.com/80
# 终端2关闭连接
[root@zhaoyl fd]# exec 8>&-
# 终端1抓包数据
11:48:49.743369 IP 172.17.233.46.37808 > 220.181.38.150.http: Flags [S], seq 117557845, win 29200, options [mss 1460,sackOK,TS val 1654376461 ecr 0,nop,wscale 7], length 0
11:48:49.748089 IP 220.181.38.150.http > 172.17.233.46.37808: Flags [S.], seq 683642453, ack 117557846, win 8192, options [mss 1452,sackOK,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,wscale 5], length 0
11:48:49.748114 IP 172.17.233.46.37808 > 220.181.38.150.http: Flags [.], ack 1, win 229, length 0


11:49:08.554855 IP 172.17.233.46.37808 > 220.181.38.150.http: Flags [F.], seq 1, ack 1, win 229, length 0
11:49:08.559783 IP 220.181.38.150.http > 172.17.233.46.37808: Flags [.], ack 2, win 916, length 0
11:49:08.559808 IP 220.181.38.150.http > 172.17.233.46.37808: Flags [F.], seq 1, ack 2, win 916, length 0
11:49:08.559821 IP 172.17.233.46.37808 > 220.181.38.150.http: Flags [.], ack 2, win 229, length 0
11:49:11.572126 IP 220.181.38.150.http > 172.17.233.46.37808: Flags [R], seq 683642455, win 0, length 0

```

握手和挥手数据：

![image-20200330115229112](E:\Repositories\docs\IO\IO模型.assets\image-20200330115229112.png)

```txt
SYN表示建立连接，
FIN表示关闭连接，
ACK表示响应，
PSH表示有 DATA数据传输，
RST表示连接重置。
```

