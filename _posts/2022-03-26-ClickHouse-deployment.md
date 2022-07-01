---
layout: post
title: ClickHouse安装部署
date: 2022-03-26 20:30:32 +0800
categories: ClickHouse
---

《ClickHouse源码编译》一文中已经讲解了源码的编译环境和编译过程，本节基于编译生成的可执行文件进行ClickHouse的安装部署。

### 1. 单机部署

#### 1）单机安装

```
[weizuo@computer ~]$ docker images               # 查看镜像
REPOSITORY                                              TAG                                 IMAGE ID            CREATED             SIZE
cr.d.xxxxxx.net/clickhouse/hello-world                       ck-v21.11.7.9-stable                23cda1419dc2        4 days ago          9.309 GB
[weizuo@computer ~]$ docker run -it --cap-add SYS_PTRACE --privileged=true --name=clickhouse-docker-test -v /home/weizuo/ClickHouse/docker:/root/clickhouse 23cda1419dc2 /bin/bash      # 启动docker容器
[root@423b4791443c ~]# cd /root/clickhouse/ClickHouse/build/programs        # 进入源码编译后生成的clickhouse可执行文件所在的目录
[root@423b4791443c programs]# ll
total 2936616
drwxr-xr-x 3 root root       4096 Dec 15 11:15 CMakeFiles
-rw-r--r-- 1 root root        677 Dec 15 11:15 CTestTestfile.cmake
drwxr-x--- 2 root root       4096 Dec 15 13:04 backups
drwxr-xr-x 4 root root       4096 Dec 15 11:15 bash-completion
drwxr-xr-x 3 root root       4096 Dec 15 12:48 benchmark
-rwxr-xr-x 1 root root 3006891832 Dec 15 12:50 clickhouse
drwxr-xr-x 3 root root       4096 Dec 15 12:48 client
-rw-r--r-- 1 root root       8304 Dec 15 11:15 cmake_install.cmake
drwxr-xr-x 3 root root       4096 Dec 15 12:44 compressor
drwxr-xr-x 3 root root       4096 Dec 15 12:48 copier
drwxr-x--- 4 root root       4096 Dec 15 13:04 data
drwxr-x--- 2 root root       4096 Dec 15 13:04 dictionaries_lib
drwxr-xr-x 3 root root       4096 Dec 15 12:43 extract-from-config
drwxr-x--- 2 root root       4096 Dec 15 13:04 flags
drwxr-xr-x 3 root root       4096 Dec 15 12:48 format
drwxr-x--- 2 root root       4096 Dec 15 13:04 format_schemas
drwxr-xr-x 3 root root       4096 Dec 15 12:44 git-import
drwxr-xr-x 3 root root       4096 Dec 15 12:44 install
drwxr-xr-x 3 root root       4096 Dec 15 12:44 keeper
drwxr-xr-x 3 root root       4096 Dec 15 12:44 keeper-converter
drwxr-xr-x 3 root root       4096 Dec 15 11:15 library-bridge
drwxr-xr-x 3 root root       4096 Dec 15 12:49 local
drwxr-x--- 4 root root       4096 Dec 15 13:04 metadata
drwxr-x--- 2 root root       4096 Dec 15 13:04 metadata_dropped
drwxr-xr-x 3 root root       4096 Dec 15 12:44 obfuscator
drwxr-xr-x 4 root root       4096 Dec 15 11:15 odbc-bridge
drwxr-x--- 2 root root       4096 Dec 16 02:35 preprocessed_configs
drwxr-xr-x 3 root root       4096 Dec 16 02:20 server
drwxr-xr-x 3 root root       4096 Dec 15 12:44 static-files-disk-uploader
drwxr-x--- 4 root root       4096 Dec 15 13:04 store
drwxr-x--- 2 root root       4096 Dec 15 13:04 tmp
drwxr-x--- 2 root root       4096 Dec 15 13:04 user_defined
drwxr-x--- 2 root root       4096 Dec 15 13:04 user_files
drwxr-x--- 2 root root       4096 Dec 15 13:04 user_scripts
-rw-r----- 1 root root         36 Dec 15 13:04 uuid
[root@423b4791443c programs]# 
[root@423b4791443c programs]# mkdir -p /usr/local/clickhouse/bin                                        # 创建可执行文件clickhouse的安装目录
[root@423b4791443c programs]# cp clickhouse /usr/local/clickhouse/bin/                           # 安装可执行文件clickhouse
[root@423b4791443c programs]# cd /root/clickhouse/ClickHouse/programs/server         # 进入配置文件模板config.xml和users.xml所在的目录
[root@423b4791443c server]# ll
total 264
-rw-r--r-- 1 root root   989 Dec 15 10:59 CMakeLists.txt
-rw-r--r-- 1 root root  4231 Dec 15 10:59 MetricsTransmitter.cpp
-rw-r--r-- 1 root root  1641 Dec 15 10:59 MetricsTransmitter.h
-rw-r--r-- 1 root root 67157 Dec 15 10:59 Server.cpp
-rw-r--r-- 1 root root  1794 Dec 15 10:59 Server.h
-rw-r--r-- 1 root root   751 Dec 15 10:59 clickhouse-server.cpp
drwxr-xr-x 2 root root  4096 Dec 15 10:59 config.d
-rw-r--r-- 1 root root 60072 Dec 15 10:59 config.xml 
-rw-r--r-- 1 root root 43239 Dec 15 10:59 config.yaml.example
-rw-r--r-- 1 root root   889 Dec 15 10:59 embedded.xml
-rw-r--r-- 1 root root 38433 Dec 15 10:59 play.html
drwxr-xr-x 2 root root  4096 Dec 15 10:59 users.d
-rw-r--r-- 1 root root  6248 Dec 15 10:59 users.xml
-rw-r--r-- 1 root root  5064 Dec 15 10:59 users.yaml.example
[root@423b4791443c server]# 
[root@423b4791443c server]# mkdir -p /usr/local/clickhouse/etc                                              # 创建文件config.xml和users.xml的安装目录
[root@423b4791443c server]# cp config.xml /usr/local/clickhouse/etc/                                  # 安装配置文件config.xml
[root@423b4791443c server]# cp users.xml /usr/local/clickhouse/etc/                                    # 安装配置文件users.xml
[root@423b4791443c server]# vi ~/.bashrc                                                                                            # 配置环境变量 export PATH=/usr/local/clickhouse/bin:$PATH
[root@423b4791443c server]# source ~/.bashrc
```

#### 2）连接测试

启动服务端：
```
[root@423b4791443c ~]# nohup clickhouse server --config-file=/usr/local/clickhouse/etc/config.xml  > /tmp/clickhouse.log 2>&1 &
```

启动客户端：
```
[root@423b4791443c ~]# clickhouse client
ClickHouse client version 21.11.7.1.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 21.11.7 revision 54450.

423b4791443c :) show databases;

SHOW DATABASES

Query id: f9ff968f-46c6-4d2b-af76-092da9eafa9e

┌─name───────────────┐
│ INFORMATION_SCHEMA │
│ default            │
│ information_schema │
│ system             │
└────────────────────┘

4 rows in set. Elapsed: 0.002 sec. 

423b4791443c :) select * from system.clusters;

SELECT *
FROM system.clusters

Query id: de1b485e-b004-479e-95f0-eab921aec3b0

┌─cluster──────────────────────────────────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name─┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┐
│ test_cluster_two_shards                      │         1 │            1 │           1 │ 127.0.0.1 │ 127.0.0.1    │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ test_cluster_two_shards                      │         2 │            1 │           1 │ 127.0.0.2 │ 127.0.0.2    │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ test_cluster_two_shards_internal_replication │         1 │            1 │           1 │ 127.0.0.1 │ 127.0.0.1    │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ test_cluster_two_shards_internal_replication │         2 │            1 │           1 │ 127.0.0.2 │ 127.0.0.2    │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ test_cluster_two_shards_localhost            │         1 │            1 │           1 │ localhost │ ::1          │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ test_cluster_two_shards_localhost            │         2 │            1 │           1 │ localhost │ ::1          │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ test_shard_localhost                         │         1 │            1 │           1 │ localhost │ ::1          │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ test_shard_localhost_secure                  │         1 │            1 │           1 │ localhost │ ::1          │ 9440 │        0 │ default │                  │            0 │               0 │                       0 │
│ test_unavailable_shard                       │         1 │            1 │           1 │ localhost │ ::1          │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ test_unavailable_shard                       │         2 │            1 │           1 │ localhost │ ::1          │    1 │        0 │ default │                  │            0 │               0 │                       0 │
└──────────────────────────────────────────────┴───────────┴──────────────┴─────────────┴───────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┘

10 rows in set. Elapsed: 0.044 sec. 

423b4791443c :) 
```
也可以使用主机和端口连接服务端：
```
[root@423b4791443c ~]# clickhouse client --host localhost --port 9000
ClickHouse client version 21.11.7.1.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 21.11.7 revision 54450.

423b4791443c :)
```
或
```
[root@423b4791443c ~]# clickhouse client --host 127.0.0.1 --port 9000
ClickHouse client version 21.11.7.1.
Connecting to 127.0.0.1:9000 as user default.
Connected to ClickHouse server version 21.11.7 revision 54450.

423b4791443c :)
```

### 2. 分布式集群部署

#### 1）Zookeeper集群安装

* 启动docker容器

选择空闲的网段，为docker创建新的网络。
```
[weizuo@computer ~]$ docker network create --subnet=172.18.0.0/24 network-wz           # 为docker创建新的网络
[weizuo@computer ~]$ docker network ls                                                                                               # 查看docker 网络
NETWORK ID          NAME                DRIVER              SCOPE
1073102225de        bridge              bridge              local               
0a7f351f599c        host                host                local               
5262239d17bd        network-wz          bridge              local               
ed023fe543f5        none                null                local 
```

基于新创建的docker网络network-wz ，启动3个docker容器clickhouse-node1、clickhouse-node2和clickhouse-node3，并分别为它们绑定固定IP。这3个docker容器用于Zookeeper集群的安装。
```
[weizuo@computer ~]$ docker run -it --cap-add SYS_PTRACE --privileged=true --network network-wz --ip 172.18.0.101 --name=clickhouse-node1 -v /home/weizuo/ClickHouse/docker:/root/clickhouse 23cda1419dc2 /bin/bash
[weizuo@computer ~]$ docker run -it --cap-add SYS_PTRACE --privileged=true --network network-wz --ip 172.18.0.102 --name=clickhouse-node2 -v /home/weizuo/ClickHouse/docker:/root/clickhouse 23cda1419dc2 /bin/bash
[weizuo@computer ~]$ docker run -it --cap-add SYS_PTRACE --privileged=true --network network-wz --ip 172.18.0.103 --name=clickhouse-node3 -v /home/weizuo/ClickHouse/docker:/root/clickhouse 23cda1419dc2 /bin/bash
```

* 在docker容器中分别安装JDK1.8

```
[root@d2a417d0f44f ~]# cd package
[root@d2a417d0f44f package]# wget .../jdk-8u311-linux-x64.tar.gz         # 下载jdk-1.8安装包
[root@d2a417d0f44f package]# tar -zxvf jdk-8u311-linux-x64.tar.gz        # 解压jdk-1.8安装包
[root@d2a417d0f44f package]# ll 
total 322928
drwxr-xr-x 18   501   501      4096 Dec 15 09:40 Python-3.7.9
-rw-r--r--  1 root  root   23277790 Aug 15  2020 Python-3.7.9.tgz
drwxrwxr-x 15 root  root       4096 Dec 15 07:37 cmake-3.20.0
-rw-r--r--  1 root  root    9427538 Mar 23  2021 cmake-3.20.0.tar.gz
drwxr-xr-x 43  1000  1000      4096 Dec 15 05:13 gcc-11.1.0
-rw-r--r--  1 root  root  138752218 Apr 27  2021 gcc-11.1.0.tar.gz
-rw-r--r--  1 root  root  146799982 Dec 18 18:46 jdk-8u311-linux-x64.tar.gz
drwxr-xr-x  8 10143 10143       273 Sep 27 12:29 jdk1.8.0_311
drwxr-xr-x  9 root  root       4096 Dec 15 07:49 ninja
drwxr-xr-x 20 root  root       4096 Dec 15 06:47 re2c
[root@d2a417d0f44f package]#
[root@d2a417d0f44f package]# mkdir -p /usr/local/java
[root@d2a417d0f44f package]# cp -r jdk1.8.0_311 /usr/local/java/            # 将jdk安装包安装在/usr/local/java/目录下
[root@d2a417d0f44f package]# vi ~/.bashrc                                                         # 配置jdk的环境变量
      export JAVA_HOME=/usr/local/java/jdk1.8.0_311
      export JRE_HOME=${JAVA_HOME}/jre
      export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
      export PATH=${JAVA_HOME}/bin:$PATH

[root@d2a417d0f44f package]# source ~/.bashrc
```

* Zookeeper集群安装

依次在每一个节点中安装zookeeper到`/usr/local/zookeeper/`目录下，并进行相应的配置。

```
[root@d2a417d0f44f ~]# cd package
[root@d2a417d0f44f package]# wget https://dlcdn.apache.org/zookeeper/zookeeper-3.7.0/apache-zookeeper-3.7.0-bin.tar.gz   # 下载Zookeeper安装包
[root@d2a417d0f44f package]# tar -zxvf apache-zookeeper-3.7.0-bin.tar.gz                                   # 解压Zookeeper安装包
[root@d2a417d0f44f package]# ll 
total 322928
drwxr-xr-x 18   501   501      4096 Dec 15 09:40 Python-3.7.9
-rw-r--r--  1 root  root   23277790 Aug 15  2020 Python-3.7.9.tgz
drwxr-xr-x  7 root  root        145 Dec 18 18:59 apache-zookeeper-3.7.0-bin
-rw-r--r--  1 root  root   12387614 Mar 27  2021 apache-zookeeper-3.7.0-bin.tar.gz
drwxrwxr-x 15 root  root       4096 Dec 15 07:37 cmake-3.20.0
-rw-r--r--  1 root  root    9427538 Mar 23  2021 cmake-3.20.0.tar.gz
drwxr-xr-x 43  1000  1000      4096 Dec 15 05:13 gcc-11.1.0
-rw-r--r--  1 root  root  138752218 Apr 27  2021 gcc-11.1.0.tar.gz
-rw-r--r--  1 root  root  146799982 Dec 18 18:46 jdk-8u311-linux-x64.tar.gz
drwxr-xr-x  8 10143 10143       273 Sep 27 12:29 jdk1.8.0_311
drwxr-xr-x  9 root  root       4096 Dec 15 07:49 ninja
drwxr-xr-x 20 root  root       4096 Dec 15 06:47 re2c
[root@d2a417d0f44f package]#
[root@d2a417d0f44f package]# mkdir -p /usr/local/zookeeper
[root@d2a417d0f44f package]# cp -r apache-zookeeper-3.7.0-bin /usr/local/zookeeper/         # 将Zookeeper安装包安装在目录/usr/local/zookeeper/下
[root@d2a417d0f44f package]# cd /usr/local/zookeeper/apache-zookeeper-3.7.0-bin/   
[root@d2a417d0f44f apache-zookeeper-3.7.0-bin]# ll
total 36
-rw-r--r-- 1 root root 11358 Dec 18 18:14 LICENSE.txt
-rw-r--r-- 1 root root   432 Dec 18 18:14 NOTICE.txt
-rw-r--r-- 1 root root  2214 Dec 18 18:14 README.md
-rw-r--r-- 1 root root  3570 Dec 18 18:14 README_packaging.md
drwxr-xr-x 2 root root  4096 Dec 18 18:14 bin
drwxr-xr-x 2 root root    92 Dec 19 10:02 conf
drwxr-xr-x 5 root root  4096 Dec 18 18:14 docs
drwxr-xr-x 2 root root  4096 Dec 18 18:14 lib
drwxr-xr-x 2 root root    75 Dec 19 10:01 logs
[root@d2a417d0f44f apache-zookeeper-3.7.0-bin]# mkdir data                                                            # 创建Zookeeper的数据目录
[root@d2a417d0f44f apache-zookeeper-3.7.0-bin]# cd conf
[root@d2a417d0f44f conf]# ll
total 16
-rw-r--r-- 1 root root  535 Dec 18 18:14 configuration.xsl
-rw-r--r-- 1 root root 3435 Dec 18 18:14 log4j.properties
-rw-r--r-- 1 root root 1148 Dec 18 18:14 zoo_sample.cfg
[root@d2a417d0f44f conf]# cp zoo_sample.cfg zoo.cfg                                                                             # 在conf目录下以zoo_sample.cfg为模板，创建配置文件zoo.cfg
[root@d2a417d0f44f conf]# vi zoo.cfg
        # 修改数据目录为刚刚创建的data目录：
        dataDir=/usr/local/zookeeper/apache-zookeeper-3.7.0-bin/data
[root@d2a417d0f44f conf]# cd ../data
[root@d2a417d0f44f data]# vi myid                                                                                                                     # 在data目录下创建新文件myid，在其中保存节点id（clickhouse-node1节点的myid保存"1"，clickhouse-node2节点的myid保存"2"，clickhouse-node3节点的myid保存"3"，注意文件中不能有任何空格和换行符）。
[root@d2a417d0f44f data]# cd ../conf/
[root@d2a417d0f44f conf]# vi zoo.cfg
        # 在zoo.cfg文件最后添加每一个zookeeper节点的配置：
        server.1=172.18.0.101:2888:3888
        server.2=172.18.0.102:2888:3888
        server.3=172.18.0.103:2888:3888

[root@d2a417d0f44f conf]# vi ~/.bashrc                                                                                                            # 配置zookeeper的环境变量 export PATH=/usr/local/zookeeper/apache-zookeeper-3.7.0-bin/bin:$PATH
[root@d2a417d0f44f conf]# source ~/.bashrc
[root@d2a417d0f44f conf]# cd
[root@d2a417d0f44f ~]# 
[root@d2a417d0f44f ~]# zkServer.sh start                                                                                                         # 在每一个zookeeper节点启动zookeeper服务
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/apache-zookeeper-3.7.0-bin/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
[root@d2a417d0f44f ~]#
```

#### 2）ClickHouse集群安装

* 启动docker容器

基于docker网络network-wz ，启动docker容器clickhouse-node4，并绑定固定IP。
```
[weizuo@computer ~]$ docker run -it --cap-add SYS_PTRACE --privileged=true --network network-wz --ip 172.18.0.104 --name=clickhouse-node4 -v /home/weizuo/ClickHouse/docker:/root/clickhouse 23cda1419dc2 /bin/bash
```
docker容器clickhouse-node1、clickhouse-node2、clickhouse-node3和clickhouse-node4位于同一网段，可用于ClickHouse集群的安装，其中，clickhouse-node1、clickhouse-node2、clickhouse-node3复用了Zookeeper集群的节点。

* ClickHouse集群安装

依次在每一个节点中安装`clickhouse`可执行文件到`/usr/local/clickhouse/bin/`目录下，并安装`config.xml`和`users.xml`到`/usr/local/clickhouse/etc/`目录下。
```
[root@d2a417d0f44f ~]# cd /root/clickhouse/ClickHouse/build/programs        # 进入源码编译后生成的clickhouse可执行文件所在的目录
[root@d2a417d0f44f programs]# ll
total 2936616
drwxr-xr-x 3 root root       4096 Dec 15 11:15 CMakeFiles
-rw-r--r-- 1 root root        677 Dec 15 11:15 CTestTestfile.cmake
drwxr-x--- 2 root root       4096 Dec 15 13:04 backups
drwxr-xr-x 4 root root       4096 Dec 15 11:15 bash-completion
drwxr-xr-x 3 root root       4096 Dec 15 12:48 benchmark
-rwxr-xr-x 1 root root 3006891832 Dec 15 12:50 clickhouse
drwxr-xr-x 3 root root       4096 Dec 15 12:48 client
-rw-r--r-- 1 root root       8304 Dec 15 11:15 cmake_install.cmake
drwxr-xr-x 3 root root       4096 Dec 15 12:44 compressor
drwxr-xr-x 3 root root       4096 Dec 15 12:48 copier
drwxr-x--- 4 root root       4096 Dec 15 13:04 data
drwxr-x--- 2 root root       4096 Dec 15 13:04 dictionaries_lib
drwxr-xr-x 3 root root       4096 Dec 15 12:43 extract-from-config
drwxr-x--- 2 root root       4096 Dec 15 13:04 flags
drwxr-xr-x 3 root root       4096 Dec 15 12:48 format
drwxr-x--- 2 root root       4096 Dec 15 13:04 format_schemas
drwxr-xr-x 3 root root       4096 Dec 15 12:44 git-import
drwxr-xr-x 3 root root       4096 Dec 15 12:44 install
drwxr-xr-x 3 root root       4096 Dec 15 12:44 keeper
drwxr-xr-x 3 root root       4096 Dec 15 12:44 keeper-converter
drwxr-xr-x 3 root root       4096 Dec 15 11:15 library-bridge
drwxr-xr-x 3 root root       4096 Dec 15 12:49 local
drwxr-x--- 4 root root       4096 Dec 15 13:04 metadata
drwxr-x--- 2 root root       4096 Dec 15 13:04 metadata_dropped
drwxr-xr-x 3 root root       4096 Dec 15 12:44 obfuscator
drwxr-xr-x 4 root root       4096 Dec 15 11:15 odbc-bridge
drwxr-x--- 2 root root       4096 Dec 16 02:35 preprocessed_configs
drwxr-xr-x 3 root root       4096 Dec 16 02:20 server
drwxr-xr-x 3 root root       4096 Dec 15 12:44 static-files-disk-uploader
drwxr-x--- 4 root root       4096 Dec 15 13:04 store
drwxr-x--- 2 root root       4096 Dec 15 13:04 tmp
drwxr-x--- 2 root root       4096 Dec 15 13:04 user_defined
drwxr-x--- 2 root root       4096 Dec 15 13:04 user_files
drwxr-x--- 2 root root       4096 Dec 15 13:04 user_scripts
-rw-r----- 1 root root         36 Dec 15 13:04 uuid
[root@d2a417d0f44f programs]# 
[root@d2a417d0f44f programs]# mkdir -p /usr/local/clickhouse/bin                                        # 创建可执行文件clickhouse的安装目录
[root@d2a417d0f44f programs]# cp clickhouse /usr/local/clickhouse/bin/                           # 安装可执行文件clickhouse
[root@d2a417d0f44f programs]# cd /root/clickhouse/ClickHouse/programs/server         # 进入配置文件模板config.xml和users.xml所在的目录
[root@d2a417d0f44f server]# ll
total 264
-rw-r--r-- 1 root root   989 Dec 15 10:59 CMakeLists.txt
-rw-r--r-- 1 root root  4231 Dec 15 10:59 MetricsTransmitter.cpp
-rw-r--r-- 1 root root  1641 Dec 15 10:59 MetricsTransmitter.h
-rw-r--r-- 1 root root 67157 Dec 15 10:59 Server.cpp
-rw-r--r-- 1 root root  1794 Dec 15 10:59 Server.h
-rw-r--r-- 1 root root   751 Dec 15 10:59 clickhouse-server.cpp
drwxr-xr-x 2 root root  4096 Dec 15 10:59 config.d
-rw-r--r-- 1 root root 60072 Dec 15 10:59 config.xml 
-rw-r--r-- 1 root root 43239 Dec 15 10:59 config.yaml.example
-rw-r--r-- 1 root root   889 Dec 15 10:59 embedded.xml
-rw-r--r-- 1 root root 38433 Dec 15 10:59 play.html
drwxr-xr-x 2 root root  4096 Dec 15 10:59 users.d
-rw-r--r-- 1 root root  6248 Dec 15 10:59 users.xml
-rw-r--r-- 1 root root  5064 Dec 15 10:59 users.yaml.example
[root@d2a417d0f44f server]# 
[root@d2a417d0f44f server]# mkdir -p /usr/local/clickhouse/etc                                              # 创建文件config.xml和users.xml的安装目录
[root@d2a417d0f44f server]# cp config.xml /usr/local/clickhouse/etc/                                  # 安装配置文件config.xml
[root@d2a417d0f44f server]# cp users.xml /usr/local/clickhouse/etc/                                    # 安装配置文件users.xml
[root@d2a417d0f44f server]# vi ~/.bashrc                                                                                            # 配置环境变量 export PATH=/usr/local/clickhouse/bin:$PATH
[root@d2a417d0f44f server]# source ~/.bashrc
```

集群配置：
```
[root@d2a417d0f44f ~]# cd /usr/local/clickhouse/etc
[root@d2a417d0f44f etc]# mkdir config.d                                                              # 创建config.d目录
[root@d2a417d0f44f etc]# ll
total 64
drwxr-xr-x 2 root root    25 Dec 19 11:37 config.d
-rw-r--r-- 1 root root 56143 Dec 19 11:37 config.xml
-rw-r--r-- 1 root root  6248 Dec 18 17:58 users.xml
[root@d2a417d0f44f etc]# cd config.d
[root@d2a417d0f44f config.d]# vi metrika.xml                                                    # 配置集群、分片和副本

          <yandex>
              <clickhouse_remote_servers>
                  <!--配置集群cluster01，可以自定义集群名称-->
                  <cluster01>
                     <!--第1个分片-->
                      <shard>
                          <internal_replication>true</internal_replication>
                          <!--第1个分片的第1个副本-->
                          <replica>
                              <host>172.18.0.101</host>
                              <port>9001</port>
                          </replica>
                          <!--第1个分片的第2个副本-->
                          <replica>
                              <host>172.18.0.102</host>
                              <port>9001</port>
                          </replica>
                      </shard>
                      <!--第2个分片-->
                      <shard>
                          <internal_replication>true</internal_replication>
                          <!--第2个分片的第1个副本-->
                          <replica>
                              <host>172.18.0.103</host>
                              <port>9001</port>
                          </replica>
                          <!--第2个分片的第2个副本-->
                          <replica>
                              <host>172.18.0.104</host>
                              <port>9001</port>
                          </replica>
                      </shard>
                  </cluster01>
              </clickhouse_remote_servers>

             <!--配置集群Zookeeper集群信息-->
              <zookeeper>
                  <node index="1">
                      <host>172.18.0.101</host>
                      <port>2181</port>
                  </node>
                  <node index="2">
                      <host>172.18.0.102</host>
                      <port>2181</port>
                  </node>
                  <node  index="3">
                      <host>172.18.0.103</host>
                      <port>2181</port>
                  </node>
              </zookeeper>

             <!--配置宏变量-->
              <macros>
                  <layer>01</layer>
                  <shard>01</shard>
                  <replica>172.18.0.101</replica>
              </macros>
          </yandex>


          注：clickhouse-node1节点的宏变量配置为：
              <macros>
                  <layer>01</layer>
                  <shard>01</shard>
                  <replica>172.18.0.101</replica>
              </macros>

          注：clickhouse-node2节点的宏变量配置为：
              <macros>
                  <layer>01</layer>
                  <shard>01</shard>
                  <replica>172.18.0.102</replica>
              </macros>

          注：clickhouse-node3节点的宏变量配置为：
              <macros>
                  <layer>01</layer>
                  <shard>02</shard>
                  <replica>172.18.0.103</replica>
              </macros>

          注：clickhouse-node4节点的宏变量配置为：
              <macros>
                  <layer>01</layer>
                  <shard>02</shard>
                  <replica>172.18.0.104</replica>
              </macros>

[root@d2a417d0f44f config.d]# cd ..
[root@d2a417d0f44f etc]# vi config.xml

    # 注释掉<remote_servers>、<macros>和<zookeeper>节点，并进行如下配置：
    <include_from>/usr/local/clickhouse/etc/config.d/metrika.xml</include_from>
    <remote_servers incl="clickhouse_remote_servers" optional="false" />
    <macros incl="macros" optional="true" />
    <zookeeper incl="zookeeper" optional="true" />

   # 设置<listen_host>节点内容如下：
   <listen_host>0.0.0.0</listen_host>
```

#### 3）连接测试

* 启动服务端

依次在每一个节点启动服务端：
```
[root@d2a417d0f44f ~]# nohup clickhouse server --config-file=/usr/local/clickhouse/etc/config.xml  > /tmp/clickhouse.log 2>&1 &
```

* 启动客户端

```
[root@d2a417d0f44f ~]# clickhouse client --host 172.18.0.104 --port 9000
ClickHouse client version 21.11.7.1.
Connecting to 172.18.0.104:9000 as user default.
Connected to ClickHouse server version 21.11.7 revision 54450.

b9c8faa3eeb9 :) select * from system.clusters;

SELECT *
FROM system.clusters

Query id: 16e1e06f-55d2-499d-acf1-72661ab6e67d

┌─cluster───┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name────┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┐
│ cluster01 │         1 │            1 │           1 │ 172.18.0.101 │ 172.18.0.101 │ 9001 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster01 │         1 │            1 │           2 │ 172.18.0.102 │ 172.18.0.102 │ 9001 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster01 │         2 │            1 │           1 │ 172.18.0.103 │ 172.18.0.103 │ 9001 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster01 │         2 │            1 │           2 │ 172.18.0.104 │ 172.18.0.104 │ 9001 │        0 │ default │                  │            0 │               0 │                       0 │
└───────────┴───────────┴──────────────┴─────────────┴──────────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┘

4 rows in set. Elapsed: 0.002 sec. 

b9c8faa3eeb9 :) 
```