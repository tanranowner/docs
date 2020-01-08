[toc]

# 1. 配置

## 2. 国内yum配置

centos7默认yum配置在/etc/yum.repos.d目录下。

```shell
[root@bogon yum.repos.d]# ll
总用量 32
-rw-r--r--. 1 root root 1664 11月 23 2018 CentOS-Base.repo
-rw-r--r--. 1 root root 1309 11月 23 2018 CentOS-CR.repo
-rw-r--r--. 1 root root  649 11月 23 2018 CentOS-Debuginfo.repo
-rw-r--r--. 1 root root  314 11月 23 2018 CentOS-fasttrack.repo
-rw-r--r--. 1 root root  630 11月 23 2018 CentOS-Media.repo
-rw-r--r--. 1 root root 1331 11月 23 2018 CentOS-Sources.repo
-rw-r--r--. 1 root root 5701 11月 23 2018 CentOS-Vault.repo
```

1. 现将这些配置进行备份。

```shell
[root@bogon yum.repos.d]# mkdir backup
[root@bogon yum.repos.d]# mv CentOS-* backup/
```

2. 下载阿里云的yum源。

```shell
[root@bogon yum.repos.d]# wget -O /etc/yum.repos.d/CenOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
--2020-01-04 13:17:19--  https://mirrors.aliyun.com/repo/Centos-7.repo
正在解析主机 mirrors.aliyun.com (mirrors.aliyun.com)... 116.177.250.229, 116.177.250.230, 116.177.250.227, ...
正在连接 mirrors.aliyun.com (mirrors.aliyun.com)|116.177.250.229|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：2523 (2.5K) [application/octet-stream]
正在保存至: “/etc/yum.repos.d/CenOS-Base.repo”

100%[===========================================================================>] 2,523       --.-K/s 用时 0s

2020-01-04 13:17:21 (395 MB/s) - 已保存 “/etc/yum.repos.d/CenOS-Base.repo” [2523/2523])

[root@bogon yum.repos.d]# ll
总用量 4
drwxr-xr-x. 2 root root  187 1月   4 13:10 backup
-rw-r--r--. 1 root root 2523 6月  16 2018 CenOS-Base.repo
```

3. 清理并重建缓存

```shell
[root@bogon yum.repos.d]# yum clean all
已加载插件：fastestmirror, langpacks
正在清理软件源： base extras updates
Cleaning up list of fastest mirrors
[root@bogon yum.repos.d]# yum makecache
已加载插件：fastestmirror, langpacks
Determining fastest mirrors
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
base                                                                                          | 3.6 kB  00:00:00
extras                                                                                        | 2.9 kB  00:00:00
updates                                                                                       | 2.9 kB  00:00:00
………………………………
(7/10): updates/7/x86_64/other_db                                                             | 368 kB  00:00:16
(8/10): base/7/x86_64/primary_db                                                              | 6.0 MB  00:00:47
(9/10): updates/7/x86_64/primary_db                                                           | 5.9 MB  00:01:00
(10/10): base/7/x86_64/filelists_db                                                           | 7.3 MB  00:01:06
元数据缓存已建立
```

## 2. 国内NTP配置

NTP 是网络时间协议 (Network Time Protocol)，它是用来同步网络中各个计算机的时间的协议。

1. 编辑/etc/ntp.conf文件

根据情况加入以下内容。

```shell
server ntp.aliyun.com iburst
```

2. 重复服务

```shell
[root@bogon yum.repos.d]# service ntpd restart
Redirecting to /bin/systemctl restart ntpd.service
[root@bogon yum.repos.d]# ntpstat
synchronised to NTP server (203.107.6.88) at stratum 3
   time correct to within 348 ms
   polling server every 64 s
```

3. 验证

```shell
[root@bogon yum.repos.d]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*203.107.6.88    10.137.55.181    2 u   33   64   77   26.295  123.785 479.286
+ntp8.flashdance 192.36.143.152   2 u   33   64   53  297.897  146.963 283.056
+undefined.hostn 127.67.113.92    2 u   33   64   57  230.755  125.787 302.799
 tick.ntp.infoma .GPS.            1 u   42   64   15  200.273  323.360 325.758
 de-user.deepini 195.13.23.5      3 u  168   64   74  339.046  120.379 133.484
[root@bogon yum.repos.d]# ping ntp.aliyun.com
PING ntp.aliyun.com (203.107.6.88) 56(84) bytes of data.
64 bytes from 203.107.6.88 (203.107.6.88): icmp_seq=1 ttl=128 time=40.1 ms
64 bytes from 203.107.6.88 (203.107.6.88): icmp_seq=2 ttl=128 time=33.0 ms
64 bytes from 203.107.6.88 (203.107.6.88): icmp_seq=3 ttl=128 time=43.7 ms
```

指令“ntpq -p”可以列出目前我们的NTP与相关的上层NTP的状态，以上的几个字段的意义如下。

remote：即NTP主机的IP或主机名称。注意最左边的符号，如果由“+”则代表目前正在作用钟的上层NTP，如果是“*”则表示也有连上线，不过是作为次要联机的NTP主机；

refid：参考的上一层NTP主机的地址；

st：即stratum阶层；

when：几秒前曾做过时间同步更新的操作；

poll：下次更新在几秒之后；

reach：已经向上层NTP服务器要求更新的次数；

delay：网络传输过程钟延迟的时间；

offset：时间补偿的结果；

jitter：Linux系统时间与BIOS硬件时间的差异时间。

## 3. 防火墙

### 1. 查看状态

查看防火墙状态命令：systemctl status firewalld

![image-20200108100157086](E:\Repositories\docs\linux\linux.assets\image-20200108100157086.png)

其中Active: active (running)就是运行并开启状态。

### 2. 关闭防火墙

关闭防火墙状态：systemctl stop firewalld.service

![image-20200108100400134](E:\Repositories\docs\linux\linux.assets\image-20200108100400134.png)

其中Active: inactive (dead)就是关闭状态。

永久关闭防火墙：systemctl disable firewalld.service

![image-20200108100658103](E:\Repositories\docs\linux\linux.assets\image-20200108100658103.png)

其中disable表示防火墙已经悠久关闭。