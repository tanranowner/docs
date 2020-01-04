[toc]

# 1. 简介

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的Linux机器上，也可以实现虚拟化，容器是完全使用沙箱机制，相互之间不会有任何接口。

docker的组成。

- 仓库（Repository）：Registry是Docker用于存放镜像文件的仓库，Docker 仓库的概念跟Git 类似。Docker Hub是Docker公司提供的一个注册服务器（Register）来保存多个仓库，每个仓库又可以包含多个具备不同tag的镜像；
- 镜像（Image）：构建容器的源代码，是一个只读的模板，由一层一层的文件系统组成的，类似于虚拟机的镜像；
- 容器（Container）：由Docker镜像创建的运行实例，类似于虚拟机，容器之间是相互隔离的，包含特定的应用及其所需的依赖文件。

# 2. 安装

## 1. 更新yum

执行yum update更新系统软件包到最新（操作有风险）。

>yum -y update：升级所有包同时也升级软件和系统内核，适用场景为需要更新内核；
>
>yum -y upgrade：只升级所有包，不升级软件和系统内核，适用场景为更新系统时，软件和内核保持原样。

## 2. 配置docker-ce镜像

```shell
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
[root@bogon yum.repos.d]# yum list docker-ce
已加载插件：fastestmirror, langpacks
Repository base is listed more than once in the configuration
Repository updates is listed more than once in the configuration
Repository extras is listed more than once in the configuration
Repository centosplus is listed more than once in the configuration
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
docker-ce-stable                                                                              | 3.5 kB  00:00:00
(1/2): docker-ce-stable/x86_64/updateinfo                                                     |   55 B  00:00:00
(2/2): docker-ce-stable/x86_64/primary_db                                                     |  37 kB  00:00:00
可安装的软件包
docker-ce.x86_64                                   3:19.03.5-3.el7                                   docker-ce-stable
```

## 3. 安装docker-ce

```shell
# Step 3: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ee.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

## 4. 安装验证

```shell
[root@bogon yum.repos.d]# docker version
Client: Docker Engine - Community
 Version:           19.03.5
 API version:       1.40
 Go version:        go1.12.12
 Git commit:        633a0ea
 Built:             Wed Nov 13 07:25:41 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.5
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.12
  Git commit:       633a0ea
  Built:            Wed Nov 13 07:24:18 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.2.10
  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
 runc:
  Version:          1.0.0-rc8+dev
  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
[root@bogon yum.repos.d]#
```

# 3. 常见命令

## 1. Docker registry

### 1. 搜索远程镜像

从远程（镜像）仓库搜索镜像。

```shell
[root@bogon yum.repos.d]# docker search mysql-57-centos7
NAME                                   DESCRIPTION                      STARS               OFFICIAL            AUTOMATED
centos/mysql-57-centos7                MySQL 5.7 SQL database server    66
jpechane/mysql-57-centos7                                               0
mskalick/mysql-57-centos7                                               0
dedyeuy/mysql-57-centos7                                                0
99cloud/mysql-57-centos7                                                0
aaron11080346/mysql-57-centos7-learn                                    0
pedro1972/mysql-57-centos7                                              0
anyt/mysql-57-centos7                                                   0
[root@bogon yum.repos.d]#
```

### 2. 拉取镜像

把远程（镜像）仓库拉取（下载）镜像到本地。

docker pull <镜像名称>

```shell
[root@bogon yum.repos.d]# docker pull mysql
Using default tag: latest
latest: Pulling from library/mysql
804555ee0376: Pull complete
c53bab458734: Pull complete
ca9d72777f90: Pull complete
2d7aad6cb96e: Pull complete
8d6ca35c7908: Pull complete
6ddae009e760: Pull complete
327ae67bbe7b: Pull complete
0e26af624120: Pull complete
5e70feb9365d: Pull complete
f5595dde544e: Pull complete
87399808d2ba: Pull complete
7312ab6d79b5: Pull complete
Digest: sha256:e1b0fd480a11e5c37425a2591b6fbd32af886bfc6d6f404bd362be5e50a2e632
Status: Downloaded newer image for mysql:latest
docker.io/library/mysql:latest
```

## 2. 镜像管理

### 1. 查看本地镜像

查看本地已经存在的镜像。

docker images [imageName]

```shell
[root@bogon yum.repos.d]# docker images mysql
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql               latest              ed1ffcb5eff3        6 days ago          456MB
[root@bogon yum.repos.d]#
```

### 2. 删除本地镜像

删除本地已经存在的镜像，如果已经存在该镜像的容器，则不能直接删除。需要先删除容器再删除镜像，或者-f强制删除。

docker rmi [imageNames]

```shell
[root@bogon yum.repos.d]# docker rmi mysql
Untagged: mysql:latest
Untagged: mysql@sha256:e1b0fd480a11e5c37425a2591b6fbd32af886bfc6d6f404bd362be5e50a2e632
Deleted: sha256:ed1ffcb5eff39aed723a66ee895854a6417485f85629de7ba87610beb6bf39ed
Deleted: sha256:a1baf24e23890cd261b3fb74574bd1ea905b0f9c6008ca304a7c1ef4e238f9fa
Deleted: sha256:13155cb001017562d45bbc41f578f8be6770163a16fd0121f03501b67e54cd86
Deleted: sha256:1cd759e6aa7d3cedd5d0c20f16bfe7e7dd91485edaf5cbf0a23bcdccc43aeea8
Deleted: sha256:e2cef815e506cd47cc6ea937f127d44d7ddf35abee7fb5ece526aaef2e48e71f
Deleted: sha256:700b12f75e2792a4111743a0aed5078c4d9b4510e68a1f10d8d2452b1d6b48de
Deleted: sha256:5f7c68324b959d2c806db18d02f153bc810f9842722415e077351bc834cc8578
Deleted: sha256:338fc0cd3fb4b87a2b83d274e8fbf475fbde19947c4ac5c5eb6e981a6fb0e8f0
Deleted: sha256:f7a4ccab931f1d1e861961eb951a7806d91ccb375e737fe1f84282f6bbafd2be
Deleted: sha256:f388e1092f8fb931a3cd07a7381bd9707d19526ff81f8b624e932f4919c27a3e
Deleted: sha256:e209b7a884b4d2e9d56bbac40ced48f2caa6a19e7ad6eb6dd20ff754f3af2c5d
Deleted: sha256:2401cf11c5455d505ef49657afcc709197ffcdfc9bd732508e9b62578a30b3a5
Deleted: sha256:814c70fdae62bc26c603bfae861f00fb1c77fc0b1ee8d565717846f4df24ae5d
[root@bogon yum.repos.d]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
[root@bogon yum.repos.d]#
```

## 3. 容器生命周期

### 1. 新建并启动容器

基于镜像创建并运行容器。

docker run [options] <image> [command] [args]

options：

-i：使用交互模式，始终保证输入流开放；

-t：分配一个伪终端，一般两个参数配合使用-it，即可在容器中利用打开的伪终端进行交互操作；

--name：为容器指定一个名称；

-p：将容器的端口暴露给宿主机，格式为-p hostport:containerport；

-v：用于挂载一个volume，可以用多个-v参数同时挂载多个volume，格式为 -v hostdir:containerdir:[rw|ro]；

-m：限制容器分配的内存总量，以B、K、M、G为单位；

-c：限制容器分配的CPU的shares值，是个相对权重，实际速度和宿主机的CPU相关。

```shell
[root@bogon yum.repos.d]# docker run -v /var/lib/docker/volumes/mysql:/var/lib/mysql -d -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 --privileged=true --name mysql mysql
d906aa8d16f86cf71ec798bf7e03fad815bb314dc0aedac0796b3787bc7d06b2
[root@bogon yum.repos.d]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
d906aa8d16f8        mysql               "docker-entrypoint.s…"   10 seconds ago      Up 9 seconds        0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
[root@bogon yum.repos.d]#

```



### 2. 停止容器

docker stop <image>

```shell
[root@bogon yum.repos.d]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
d906aa8d16f8        ed1ffcb5eff3        "docker-entrypoint.s…"   6 minutes ago       Up 6 minutes        0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
[root@bogon yum.repos.d]# docker stop mysql
mysql
[root@bogon yum.repos.d]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
d906aa8d16f8        ed1ffcb5eff3        "docker-entrypoint.s…"   7 minutes ago       Exited (0) 7 seconds ago                       mysql
```

### 3. 重启容器

docker restart <container>

### 4. 启动容器

docker start <container>

### 5. 删除容器

docker rm <container>

```shell
[root@bogon yum.repos.d]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
d906aa8d16f8        ed1ffcb5eff3        "docker-entrypoint.s…"   7 minutes ago       Exited (0) 9 seconds ago                       mysql
[root@bogon yum.repos.d]# docker rm mysql
mysql
[root@bogon yum.repos.d]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```



## 4. 运维操作

### 1. 查看容器信息

docker ps

```shell
[root@bogon yum.repos.d]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
8d65b8922b4e        mysql:5.7           "docker-entrypoint.s…"   2 minutes ago       Up About a minute   0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
```

### 2. 查看容器和镜像详细信息

docker inspect <container>

```shell
[root@bogon yum.repos.d]# docker inspect mysql
[
    {
        "Id": "8d65b8922b4e19ec2ea7d69b6955f9ff585e7b57baad76ce25524faab3bda02b",
        "Created": "2020-01-04T09:41:57.451615037Z",
        "Path": "docker-entrypoint.sh",
        "Args": [
            "mysqld"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 78502,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2020-01-04T09:43:40.039666226Z",
            "FinishedAt": "2020-01-04T09:41:58.349456248Z"
        },
        "Image": "sha256:db39680b63ac47a1d989da7b742f7b382af34d85a68214f8977bad59c05901a6",
        "ResolvConfPath": "/var/lib/docker/containers/8d65b8922b4e19ec2ea7d69b6955f9ff585e7b57baad76ce25524faab3bda02b/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/8d65b8922b4e19ec2ea7d69b6955f9ff585e7b57baad76ce25524faab3bda02b/hostname",
        "HostsPath": "/var/lib/docker/containers/8d65b8922b4e19ec2ea7d69b6955f9ff585e7b57baad76ce25524faab3bda02b/hosts",
        "LogPath": "/var/lib/docker/containers/8d65b8922b4e19ec2ea7d69b6955f9ff585e7b57baad76ce25524faab3bda02b/8d65b8922b4e19ec2ea7d69b6955f9ff585e7b57baad76ce25524faab3bda02b-json.log",
        "Name": "/mysql",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": [
                "/var/lib/docker/volumes/mysql:/var/lib/mysql"
            ],
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {
                "3306/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "3306"
                    }
                ]
            },
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "Capabilities": null,
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": true,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": [
                "label=disable"
            ],
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "KernelMemory": 0,
            "KernelMemoryTCP": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": null,
            "ReadonlyPaths": null
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/1bd1e1d8af0602be5fe562732b5330b74c3020db1e39fc38c4c9d0ac89445ee9-init/diff:/var/lib/docker/overlay2/5d73b1dd7e54559536beedd591c0dd36e399f89786a01678fa2d4f919e22361f/diff:/var/lib/docker/overlay2/ef263e3677b21c128a13481b7990beb7c8a0cc78243bfc170e183ec8075956ed/diff:/var/lib/docker/overlay2/b05a21271423892ce34bf4fbe3c4a205bb658ef48032959b2e762caf850aacdb/diff:/var/lib/docker/overlay2/5bbcbca3a5c366616ae19aa8f36ad4ba02ed1f96617a99c0baca58addf57a289/diff:/var/lib/docker/overlay2/4fb7edb96530da4c22fb7474f1c3bf2683e619d1f83d65c9ddfbc0cfbce51c52/diff:/var/lib/docker/overlay2/da6a818545f78b81c5fa0255ef7e98911bb545b513357667b920b9a50bf1611a/diff:/var/lib/docker/overlay2/6d52fe7a157831a1b34e0f9422e258447e41604da0c0f4314509ad60a8926165/diff:/var/lib/docker/overlay2/c02fa997ec6bd8259d1cf7e764e971fc8e6451e481b6afcd9cceaa02e5908de5/diff:/var/lib/docker/overlay2/f94f2f1016f91259b0d9d9ae3bbde4888f134642a1ea66527b6b4bc00d536787/diff:/var/lib/docker/overlay2/0d5a39027a51be95dfd4dd2615c10756607413eceffe1802fb9fccd44c52e56b/diff:/var/lib/docker/overlay2/ece6152cc2473db8f8d8952acc6039c9216336de7e1e021ac8f7582897c8f5c6/diff",
                "MergedDir": "/var/lib/docker/overlay2/1bd1e1d8af0602be5fe562732b5330b74c3020db1e39fc38c4c9d0ac89445ee9/merged",
                "UpperDir": "/var/lib/docker/overlay2/1bd1e1d8af0602be5fe562732b5330b74c3020db1e39fc38c4c9d0ac89445ee9/diff",
                "WorkDir": "/var/lib/docker/overlay2/1bd1e1d8af0602be5fe562732b5330b74c3020db1e39fc38c4c9d0ac89445ee9/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/var/lib/docker/volumes/mysql",// 宿主机挂载点
                "Destination": "/var/lib/mysql",//容器挂载点
                "Mode": "",
                "RW": true,
                "Propagation": "rslave"
            }
        ],
        "Config": {
            "Hostname": "8d65b8922b4e",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "3306/tcp": {},
                "33060/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "MYSQL_ROOT_PASSWORD=123456",
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "GOSU_VERSION=1.7",
                "MYSQL_MAJOR=5.7",
                "MYSQL_VERSION=5.7.28-1debian9"
            ],
            "Cmd": [
                "mysqld"
            ],
            "Image": "mysql:5.7",
            "Volumes": {
                "/var/lib/mysql": {}
            },
            "WorkingDir": "",
            "Entrypoint": [
                "docker-entrypoint.sh"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "9b2a7ec93678544f43873bf12e8528c9006401e8c47c0f4976496dd7828dd359",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "3306/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "3306"
                    }
                ],
                "33060/tcp": null
            },
            "SandboxKey": "/var/run/docker/netns/9b2a7ec93678",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "4999f446689ebc63b0705f20933e21bd287e9f250dd91b16638ef5a479506e14",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:02",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "b463ea5e794af901471282ea87f73bc42d2be1a2dcf335417831d0c13d1ca661",
                    "EndpointID": "4999f446689ebc63b0705f20933e21bd287e9f250dd91b16638ef5a479506e14",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
[root@bogon yum.repos.d]# 
```



### 3. 连接容器

docker attach <container>

### 4. 进入容器

docker exec <container> [arg]

```shell
[root@bogon ~]# docker exec -it 8d /bin/bash
root@8d65b8922b4e:/# ls
bin   dev                         entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint-initdb.d  etc            lib   media  opt  root  sbin  sys  usr
root@8d65b8922b4e:/# pwd
/
```

### 5. 容器日志

docker logs <container>

```shell
[root@bogon ~]# docker logs mysql
2020-01-04 09:41:57+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.7.28-1debian9 started.
2020-01-04 09:41:57+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2020-01-04 09:41:57+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.7.28-1debian9 started.
2020-01-04T09:41:58.212200Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2020-01-04T09:41:58.213980Z 0 [Note] mysqld (mysqld 5.7.28) starting as process 1 ...
2020-01-04T09:41:58.217448Z 0 [Note] InnoDB: PUNCH HOLE support available
2020-01-04T09:41:58.217480Z 0 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2020-01-04T09:41:58.217483Z 0 [Note] InnoDB: Uses event mutexes
2020-01-04T09:41:58.217485Z 0 [Note] InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
2020-01-04T09:41:58.217487Z 0 [Note] InnoDB: Compressed tables use zlib 1.2.11
2020-01-04T09:41:58.217490Z 0 [Note] InnoDB: Using Linux native AIO
2020-01-04T09:41:58.217759Z 0 [Note] InnoDB: Number of pools: 1
2020-01-04T09:41:58.217904Z 0 [Note] InnoDB: Using CPU crc32 instructions
2020-01-04T09:41:58.219823Z 0 [Note] InnoDB: Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
…………
```



# 4. Dockerfile

Dockerfile是Docker用来构建镜像的文本文件，包含自定义的指令和格式，可以通过docker build命令从Dockerfile中构建镜像。用户可以通过这些统一的语法命令来根据需求进行配置，通过这份统一的配置文件，在不同的平台上进行分发，需要使用时就可以使用配置文件自动化构建。同时，Dockerfile与镜像配合使用，是Docker在构建时可以充分利用镜像的功能进行缓存，提升了Docker的使用效率。

## 1. 指令

Dockerfile的基本格式如下：

```shell
#Comment
INSTRUCTION arguments
```

指令INSTRUCTION不区分大小写，但是为了和参数区分，建议大写。Docker会按照书写顺序逐行执行，第一行指令必须为FROM，用来指定构建镜像的基础镜像。以#开头的行为注释行，非以#开头的则为普通的参数。

Dockerfile中的指令有FROM、MAINTAINER、RUN、CMD、EXPOSE、ENV、ADD、COPY、ENTRYPOINT、VOLUME、USER、WORKDIR、ONBUILD，如果遇到错误的指令会被忽略。

### 1. FROM

指定构建的基础镜像。

FROM <image>:[tag]

### 2. ENV

指定镜像容器的环境变量。ENV指令声明的环境变量会被后面的特定指令（ENV、COPY、WORKDIR、EXPOSE、VOLUME、USER）使用。其它指令使用环境变量时，使用格式为$variable_name或者${variable_name}。在变量前面加斜杠\可以转移为普通字符串，如\\$foo，则为普通的字符串***$foo***，而不会被替换为变量值。

>ONBUILD指令不支持环境变量。

ENV <key> <value>或者ENV <key>=<value>

### 3. COPY

COPY <src> <dest>

COPY指令复制src文件或者目录，将它添加到新镜像中的dest目录。src必须是上下文根目录的相对路径，可以使用通配符。src和dest如果以/结尾表示目录，否则为文件。dest必须是镜像中的绝对路径或者相对于WORKDIR的相对路径。

### 4. ADD

ADD <src> <dest>

ADD和COPY指令类似，但ADD还支持其它功能。如果src是个压缩文件，则复制到镜像红时会自动解压。

### 5. RUN

shell格式：RUN <command>

exec格式：RUN ["executable","param1","param2"]

RUN指令会在前一条命令创建出的镜像基础上创建一个容器，并且容器启动时运行此命令，在命令结束运行后提交容器为新镜像，新镜像被Dockerfile中的下一条指令使用。

当使用shell格式时，命令是通过/bin/bash -c运行；当使用exec格式时，命令是直接运行而不调用shell程序。exec格式中的参数会被Docker以json数组解析，由于没有调用shell，所有环境变量不会被替换。

### 6. CMD

shell格式：CMD <command>

exec格式：CMD ["executable","param1","param2"]

ENTRYPOINT参数格式：CMD ["param1","param2"]

CMD提供容器运行时的默认值，这些默认值是一条指令或者一些参数。一个Dockerfile中可以有多条CMD命令，但是只有最后一条CMD才有效。CMD中的参数会添加到ENTRYPOINT指令中。使用shell格式时，和RUN的运行方式是一样的。不同的是，RUN指令会在构建镜像时执行，并生成新的镜像；CMD在构建镜像时并不执行，而是在容器启动时默认将CMD作为第一条执行的命令。如果用户在命令行运行docker run时至此那个了命令参数会覆盖CMD中的命令。

### 7. ENTRYPOINT

shell格式：ENTRYPOINT <command>

exec格式：ENTRYPOINT ["executable","param1","param2"]

ENTRYPOINT和CMD指令类似，都可以让容器启动时执行。不同的是，一个Dockerfile中可以有多个ENTRYPOINT指令，但是只有最后一条生效。当以shell格式执行时，ENTRYPOINT会忽略CMD和docker run后面的参数；当以exec格式执行时，docker run传入的命令参数会覆盖CMD中的内容并附加到ENTRYPOINT指令的参数中。ENTRYPOINT只能是命令，CMD可以是命令或者参数，另外，docker run的命令行参数可以覆盖CMD，但是不能覆盖ENTRYPOINT。

### 8. 样例

```shell
#第一行必须是FROM
FROM java:8
LABEL email=xxx@gmail.com
ADD eureka-0.0.1-SNAPSHOT.jar /opt/eureka-0.0.1-SNAPSHOT.jar
EXPOSE 5555
RUN mkdir -p /opt/logs
CMD java -jar /opt/eureka-0.0.1-SNAPSHOT.jar > /opt/logs/eureka.log

docker build -t eureka:0.0.1 .
```

