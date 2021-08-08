---
layout: post
title: ClickHouse编译
categories: ClickHouse
description: ClickHouse编译
keywords: ClickHouse, Build
---

## ClickHouse源码编译

### 1. 准备系统

本文采用docker容器环境进行ClickHouse源码编译的编译和测试。

#### 1）下载镜像

```
[weizuo@computer ~]$ docker pull centos:7         #下载CentOS 7的docker镜像
Trying to pull repository docker.io/library/centos ... 
7: Pulling from docker.io/library/centos
2d473b07cdd5: Pull complete 
Digest: sha256:9d4bcbbb213dfd745b58be38b13b996ebb5ac315fe75711bd618426a630e0987
[weizuo@computer ~]$
[weizuo@computer ~]$ docker images                      # 查看下载的镜像
REPOSITORY   
docker.io/centos                                        7                                   eeb6ee3f44bd        3 months ago        203.9 MB
[weizuo@computer ~]$ 
```

#### 2）启动容器

```
[weizuo@computer ~]$ docker run -it --cap-add SYS_PTRACE --privileged=true --name=clickhouse-build-environment -v /home/weizuo/ClickHouse/docker:/root/clickhouse eeb6ee3f44bd /bin/bash    # 启动容器
[root@b34940c19b29 /]# cd root/
[root@b34940c19b29 ~]# ll
total 8
-rw------- 1 root  root  3416 Nov 13  2020 anaconda-ks.cfg
drwxrwxr-x 2 10097 10098 4096 Dec 15 04:09 clickhouse
[root@b34940c19b29 ~]# 
[root@b34940c19b29 ~]# cat /etc/redhat-release                         # 查看系统版本
CentOS Linux release 7.9.2009 (Core)
[root@b34940c19b29 ~]# 
```

#### 3）安装依赖项

```
[root@b34940c19b29 ~]# yum update                                                                                                                                                                    #更新的package更新到源中的最新版
[root@b34940c19b29 ~]# yum -y install gcc gcc-c++ wget bzip2 automake autoconf libtool make  git openssl-devel zlib-devel bzip2-devel ncurses-devel sqlite-devel readline-devel tk-devel python-pip libffi gdbm-devel xz-devel lzma libffi-devel      # 安装低版本gcc和g++（用于编译高版本的GCC）等依赖项
[root@b34940c19b29 ~]# gcc -v                                                                                                                                                                                 # 查看gcc版本
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
Thread model: posix
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
[root@b34940c19b29 ~]# 
[root@b34940c19b29 ~]# g++ -v                                                                                                                                                                                  # 查看g++版本
Using built-in specs.
COLLECT_GCC=g++
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
Thread model: posix
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
[root@b34940c19b29 ~]# 
```

### 2.安装GCC-11.1.0

#### 1）下载安装包

```
[root@b34940c19b29 ~]# ll
total 8
-rw------- 1 root  root  3416 Nov 13  2020 anaconda-ks.cfg
drwxrwxr-x 2 10097 10098 4096 Dec 15 04:09 clickhouse
[root@b34940c19b29 ~]#
[root@b34940c19b29 ~]# mkdir package                                                                                                                # 创建package目录，用于保存下载的安装包
[root@b34940c19b29 ~]# ll
total 8
-rw------- 1 root  root  3416 Nov 13  2020 anaconda-ks.cfg
drwxrwxr-x 2 10097 10098 4096 Dec 15 04:09 clickhouse
drwxr-xr-x 2 root  root     6 Dec 15 04:57 package
[root@b34940c19b29 ~]# cd package/                                                                                                                       # 进入package目录
[root@b34940c19b29 package]# wget http://ftp.gnu.org/gnu/gcc/gcc-11.1.0/gcc-11.1.0.tar.gz      # 下载GCC-11安装包gcc-11.1.0.tar.gz
[root@b34940c19b29 package]# 
[root@b34940c19b29 package]# 
[root@b34940c19b29 package]# ll
total 135504
-rw-r--r-- 1 root root 138752218 Apr 27  2021 gcc-11.1.0.tar.gz
[root@b34940c19b29 package]#
```

#### 2）编译及安装

```
[root@b34940c19b29 package]# tar -zxvf gcc-11.1.0.tar.gz                                                                               # 解压下载的GCC-11安装包
[root@b34940c19b29 package]#
[root@b34940c19b29 package]# ll
total 135508
drwxr-xr-x 38 1000 1000      4096 Apr 27  2021 gcc-11.1.0
-rw-r--r--  1 root root 138752218 Apr 27  2021 gcc-11.1.0.tar.gz
[root@b34940c19b29 package]# cd gcc-11.1.0                                                                                                        # 进入gcc-11.1.0目录
[root@b34940c19b29 gcc-11.1.0]#
[root@b34940c19b29 gcc-11.1.0]# 
[root@b34940c19b29 gcc-11.1.0]# 
[root@b34940c19b29 gcc-11.1.0]# ./contrib/download_prerequisites                                                          # 下载编译gcc-11.1.0所需的依赖项
2021-12-15 05:06:40 URL:http://gcc.gnu.org/pub/gcc/infrastructure/gmp-6.1.0.tar.bz2 [2383840/2383840] -> "gmp-6.1.0.tar.bz2" [1]
2021-12-15 05:07:48 URL:http://gcc.gnu.org/pub/gcc/infrastructure/mpfr-3.1.4.tar.bz2 [1279284/1279284] -> "mpfr-3.1.4.tar.bz2" [1]
2021-12-15 05:08:18 URL:http://gcc.gnu.org/pub/gcc/infrastructure/mpc-1.0.3.tar.gz [669925/669925] -> "mpc-1.0.3.tar.gz" [1]
2021-12-15 05:10:48 URL:http://gcc.gnu.org/pub/gcc/infrastructure/isl-0.18.tar.bz2 [1658291/1658291] -> "isl-0.18.tar.bz2" [1]
gmp-6.1.0.tar.bz2: OK
mpfr-3.1.4.tar.bz2: OK
mpc-1.0.3.tar.gz: OK
isl-0.18.tar.bz2: OK
All prerequisites downloaded successfully.
[root@b34940c19b29 gcc-11.1.0]#
[root@b34940c19b29 gcc-11.1.0]# mkdir build                                                                                                             # 创建build目录，用于编译gcc-11.1.0
[root@b34940c19b29 gcc-11.1.0]# cd build/
[root@b34940c19b29 build]# mkdir /usr/lib/gcc/x86_64-redhat-linux/11.1.0                                                 # 创建gcc-11.1.0的安装目录
[root@b34940c19b29 build]#
[root@b34940c19b29 build]# ../configure --prefix=/usr/lib/gcc/x86_64-redhat-linux/11.1.0  --enable-bootstrap  --enable-checking=release --enable-languages=c,c++ --disable-multilib              # 执行编译配置
[root@b34940c19b29 build]# make -j 20                                                                                                                           # 执行编译，编译并发设置为20
[root@b34940c19b29 build]#
[root@b34940c19b29 build]# make install                                                                                                                       # 执行安装
```

#### 3）GCC版本配置

```
[root@b34940c19b29 build]# mv /usr/bin/gcc /usr/bin/gcc-4.8.5                                    # 设置系统之前的gcc版本为gcc-4.8.5
[root@b34940c19b29 build]# mv /usr/bin/g++ /usr/bin/g++-4.8.5                                   # 设置系统之前的g++版本为g++-4.8.5
[root@b34940c19b29 build]# alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8.5 88 --slave /usr/bin/g++ g++ /usr/bin/g++-4.8.5                                                                                                                                                                                                                                   # 通过alternatives命令配置gcc-4.8.5为可选择的gcc版本
[root@b34940c19b29 build]# alternatives --install /usr/bin/gcc gcc /usr/lib/gcc/x86_64-redhat-linux/11.1.0/bin/x86_64-pc-linux-gnu-gcc 99 --slave /usr/bin/g++ g++ /usr/lib/gcc/x86_64-redhat-linux/11.1.0/bin/x86_64-pc-linux-gnu-g++                 # 通过alternatives命令配置gcc-11.1.0为可选择的gcc版本
[root@b34940c19b29 build]# alternatives --config gcc                                                         # 配置系统启动的gcc版本

There are 2 programs which provide 'gcc'.

  Selection    Command
-----------------------------------------------
   1           /usr/bin/gcc-4.8.5
*+ 2           /usr/lib/gcc/x86_64-redhat-linux/11.1.0/bin/x86_64-pc-linux-gnu-gcc

Enter to keep the current selection[+], or type selection number: 2                                 # 配置系统启动的gcc版本为gcc-11.1.0
[root@b34940c19b29 build]# 
[root@b34940c19b29 build]# rm -f /usr/lib64/libstdc++.so.6
[root@b34940c19b29 build]# ln -s /usr/lib/gcc/x86_64-redhat-linux/11.1.0/lib64/libstdc++.so.6 /usr/lib64/libstdc++.so.6       # 更新libstdc++库
[root@b34940c19b29 build]# rm -rf /lib64/libstdc++.so.6
[root@b34940c19b29 build]# ln -s /usr/lib/gcc/x86_64-redhat-linux/11.1.0/lib64/libstdc++.so.6 /lib64/libstdc++.so.6                # 更新libstdc++库
[root@b34940c19b29 build]# 
[root@b34940c19b29 build]# strings /usr/lib64/libstdc++.so.6 | grep GLIBC                  # 查看libstdc++库中的版本
[root@b34940c19b29 build]# 
[root@b34940c19b29 build]# gcc -v                                                                                                  # 查看gcc版本
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-redhat-linux/11.1.0/libexec/gcc/x86_64-pc-linux-gnu/11.1.0/lto-wrapper
Target: x86_64-pc-linux-gnu
Configured with: ../configure --prefix=/usr/lib/gcc/x86_64-redhat-linux/11.1.0 --enable-bootstrap --enable-checking=release --enable-languages=c,c++ --disable-multilib
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 11.1.0 (GCC) 
[root@b34940c19b29 build]# 
[root@b34940c19b29 build]# g++ -v                                                                                                  # 查看g++版本
Using built-in specs.
COLLECT_GCC=g++
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-redhat-linux/11.1.0/libexec/gcc/x86_64-pc-linux-gnu/11.1.0/lto-wrapper
Target: x86_64-pc-linux-gnu
Configured with: ../configure --prefix=/usr/lib/gcc/x86_64-redhat-linux/11.1.0 --enable-bootstrap --enable-checking=release --enable-languages=c,c++ --disable-multilib
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 11.1.0 (GCC) 
[root@b34940c19b29 build]#
```

### 3.安装re2c

```
[root@b34940c19b29 package]# git clone https://github.com/skvadrik/re2c.git                   # 下载re2c包
[root@b34940c19b29 package]# ll
total 135512
drwxr-xr-x 43 1000 1000      4096 Dec 15 05:13 gcc-11.1.0
-rw-r--r--  1 root root 138752218 Apr 27  2021 gcc-11.1.0.tar.gz
drwxr-xr-x 16 root root      4096 Dec 15 06:44 re2c
[root@b34940c19b29 package]# cd re2c
[root@b34940c19b29 re2c]#
[root@b34940c19b29 re2c]# mkdir -p m4
[root@b34940c19b29 re2c]# ./autogen.sh
[root@b34940c19b29 re2c]# ./configure --prefix=/usr                                                                           # 编译配置
[root@b34940c19b29 re2c]# make -j 10                                                                                                       # 执行编译
[root@b34940c19b29 re2c]# make install                                                                                                   # 执行安装
[root@b34940c19b29 re2c]# 
[root@b34940c19b29 re2c]# re2c --version                                                                                                # 查看re2c版本
re2c 2.2
[root@b34940c19b29 re2c]# 
```

### 4.安装CMake-3.20.0

```
[root@b34940c19b29 package]# wget https://github.com/Kitware/CMake/releases/download/v3.20.0/cmake-3.20.0.tar.gz    # 下载cmake-3.20.0.tar.gz包
[root@b34940c19b29 package]# ll
total 144720
-rw-r--r--  1 root root   9427538 Mar 23  2021 cmake-3.20.0.tar.gz
drwxr-xr-x 43 1000 1000      4096 Dec 15 05:13 gcc-11.1.0
-rw-r--r--  1 root root 138752218 Apr 27  2021 gcc-11.1.0.tar.gz
drwxr-xr-x 20 root root      4096 Dec 15 06:47 re2c
[root@b34940c19b29 package]# tar -zxvf cmake-3.20.0.tar.gz                                                                         # 解压cmake-3.20.0.tar.gz包
[root@b34940c19b29 package]# cd cmake-3.20.0
[root@b34940c19b29 cmake-3.20.0]# ./bootstrap --prefix=/usr/local/cmake3-20                                   # 编译配置
[root@b34940c19b29 package]# gmake -j 10                                                                                                           # 执行编译
[root@b34940c19b29 cmake-3.20.0]# gmake install                                                                                             # 执行安装
[root@b34940c19b29 cmake-3.20.0]# vi ~/.bashrc                                                                                                 # 配置环境变量 export PATH=/usr/local/cmake3-20/bin:$PATH
[root@b34940c19b29 cmake-3.20.0]# source ~/.bashrc 
[root@b34940c19b29 cmake-3.20.0]# cmake --version                                                                                        # 查看cmake版本
cmake version 3.20.0
CMake suite maintained and supported by Kitware (kitware.com/cmake).
[root@b34940c19b29 cmake-3.20.0]# 
```

### 5.安装ninja

```
[root@b34940c19b29 package]# git clone https://github.com/ninja-build/ninja.git  # 下载ninja包
[root@b34940c19b29 package]# ll
total 144724
drwxrwxr-x 15 root root      4096 Dec 15 07:37 cmake-3.20.0
-rw-r--r--  1 root root   9427538 Mar 23  2021 cmake-3.20.0.tar.gz
drwxr-xr-x 43 1000 1000      4096 Dec 15 05:13 gcc-11.1.0
-rw-r--r--  1 root root 138752218 Apr 27  2021 gcc-11.1.0.tar.gz
drwxr-xr-x  8 root root       295 Dec 15 07:47 ninja
drwxr-xr-x 20 root root      4096 Dec 15 06:47 re2c
[root@b34940c19b29 package]# 
[root@b34940c19b29 package]# cd ninja
[root@b34940c19b29 ninja]# ./configure.py --bootstrap                                                           # 编译ninja
[root@b34940c19b29 ninja]# cp ninja /usr/bin/                                                                             # 将ninja可执行文件复制到/usr/bin/目录下
[root@b34940c19b29 ninja]# ninja --version                                                                                   # 查看ninja版本
1.10.2.git
[root@b34940c19b29 ninja]#
```

### 6. 安装Python3

```
[root@b34940c19b29 package]# python --version                                                                                                              # 查看系统默认的python版本
Python 2.7.5
[root@b34940c19b29 package]# mv /usr/bin/python /usr/bin/python2.7.5                                                            # 将系统默认python的可执行文件更改为python2.7.5
[root@b34940c19b29 package]# wget https://www.python.org/ftp/python/3.7.9/Python-3.7.9.tgz            # 下载Python-3.7.9安装包
[root@b34940c19b29 package]# tar -zxvf Python-3.7.9.tgz                                                                                              # 解压Python-3.7.9安装包
[root@b34940c19b29 package]# ll
total 167468
drwxr-xr-x 17  501  501      4096 Aug 15  2020 Python-3.7.9
-rw-r--r--  1 root root  23277790 Aug 15  2020 Python-3.7.9.tgz
drwxrwxr-x 15 root root      4096 Dec 15 07:37 cmake-3.20.0
-rw-r--r--  1 root root   9427538 Mar 23  2021 cmake-3.20.0.tar.gz
drwxr-xr-x 43 1000 1000      4096 Dec 15 05:13 gcc-11.1.0
-rw-r--r--  1 root root 138752218 Apr 27  2021 gcc-11.1.0.tar.gz
drwxr-xr-x  9 root root      4096 Dec 15 07:49 ninja
drwxr-xr-x 20 root root      4096 Dec 15 06:47 re2c
[root@b34940c19b29 package]# cd Python-3.7.9
[root@b34940c19b29 Python-3.7.9]# ./configure prefix=/usr/local/python3                                                              # 编译配置
[root@b34940c19b29 Python-3.7.9]# make -j 10                                                                                                                      # 执行编译
[root@b34940c19b29 Python-3.7.9]# make install                                                                                                                  # 执行安装
[root@b34940c19b29 Python-3.7.9]# ln -s /usr/local/python3/bin/python3.7 /usr/bin/python                         # 创建软连接将新安装的python3.7作为系统默认的python版本


[root@b34940c19b29 package]# vi /usr/bin/yum                                                    # 修改yum配置，修改文件首行的#! /usr/bin/python为#! /usr/bin/python2
[root@b34940c19b29 package]# vi /usr/libexec/urlgrabber-ext-down           # 修改文件首行的#! /usr/bin/python为#! /usr/bin/python2
```

### 7. 下载并编译ClickHouse

```
[root@b34940c19b29 clickhouse]# git config --global url."https://hub.fastgit.org".insteadOf https://github.com         # 替换github的源
[root@b34940c19b29 clickhouse]# git clone https://github.com/ClickHouse/ClickHouse.git                                                  # 下载ClickHouse源码
[root@b34940c19b29 clickhouse]# cd ClickHouse
[root@b34940c19b29 ClickHouse]# git tag -l                                                                                                                                                   # 查看仓库tag列表
53973
53974
53975
53976
53977
53978
53979
53980

                 ...

v21.10.1.7829-testing
v21.10.1.7845-testing
v21.10.1.7848-testing
v21.10.1.7856-testing
v21.10.1.7859-testing
v21.10.1.7892-testing
v21.10.1.7906-testing
v21.10.1.7926-testing
v21.10.1.7933-testing
v21.10.1.7939-testing
v21.10.1.7998-testing
v21.10.1.8002-testing
v21.10.1.8009-testing
v21.10.1.8013-prestable
v21.10.1.8013-testing
v21.10.2.15-stable
v21.10.3.9-stable
v21.10.4.26-stable
v21.10.5.3-stable
v21.11.1.8636-prestable
v21.11.2.2-stable
v21.11.3.6-stable
v21.11.4.14-stable
v21.11.5.33-stable
v21.11.6.7-stable
v21.11.7.9-stable
                 ...
[root@b34940c19b29 ClickHouse]#
[root@b34940c19b29 build]# git checkout -b branch-v21.11.7.9-stable v21.11.7.9-stable                                            # 创建新分支并切换到最新的稳定版v21.11.7.9-stable
[root@b34940c19b29 ClickHouse]# git branch -vv
* branch-v21.11.7.9-stable 2cb868a Merge pull request #32711 from ClickHouse/backport/21.11/32359
  master                   ebdf4d2 [origin/master] Merge pull request #32796 from ClickHouse/fix_performance_build_path
  [root@b34940c19b29 ClickHouse]#
[root@b34940c19b29 ClickHouse]# git submodule sync
[root@b34940c19b29 ClickHouse]# git submodule update --init --recursive                                                                                    # 下载需要的第三方包
[root@b34940c19b29 ClickHouse]# mkdir build
[root@b34940c19b29 ClickHouse]# cd build/
[root@b34940c19b29 build]#
[root@b34940c19b29 build]# vi ~/.bashrc                                                                                                                                                         # 配置环境变量export C=gcc和export CXX=g++
[root@b34940c19b29 build]# source ~/.bashrc
[root@b34940c19b29 build]#
[root@b34940c19b29 build]# cmake -DCMAKE_INSTALL_PREFIX=/usr/local/clickhouse ../                                                        # 执行cmake进行编译配置
[root@b34940c19b29 build]# ninja clickhouse                                                                                                                                                # 执行编译
```

* 执行cmake命令可能会报错：
```
cmake: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.29' not found (required by cmake)
cmake: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.22' not found (required by cmake)
cmake: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.20' not found (required by cmake)
cmake: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.26' not found (required by cmake)
cmake: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.21' not found (required by cmake)
cmake: /lib64/libstdc++.so.6: version `CXXABI_1.3.9' not found (required by cmake)
```
解决：执行以下命令后重新执行cmake命令：
```
[root@b34940c19b29 build]# rm -rf /lib64/libstdc++.so.6
[root@b34940c19b29 build]# ln -s /usr/lib/gcc/x86_64-redhat-linux/11.1.0/lib64/libstdc++.so.6 /lib64/libstdc++.so.6
```

* 编译过程中可能会报错：
```
FAILED: base/glibc-compatibility/CMakeFiles/glibc-compatibility.dir/musl/eventfd.c.o 
/usr/bin/gcc  -I../base/glibc-compatibility/libcxxabi -I../base/glibc-compatibility/musl/x86_64 -fdiagnostics-color=always -include /root/clickhouse/ClickHouse/base/glibc-compatibility/glibc-compat-2.32.h  -pipe -mssse3 -msse4.1 -msse4.2 -mpclmul -mpopcnt -fasynchronous-unwind-tables -ffile-prefix-map=/root/clickhouse/ClickHouse=. -falign-functions=32  -Wall  -Werror -fomit-frame-pointer -O2 -g -DNDEBUG -O3    -D OS_LINUX -Wno-unused-command-line-argument -Wno-unused-but-set-variable -Wno-builtin-requires-header -std=gnu11 -MD -MT base/glibc-compatibility/CMakeFiles/glibc-compatibility.dir/musl/eventfd.c.o -MF base/glibc-compatibility/CMakeFiles/glibc-compatibility.dir/musl/eventfd.c.o.d -o base/glibc-compatibility/CMakeFiles/glibc-compatibility.dir/musl/eventfd.c.o -c ../base/glibc-compatibility/musl/eventfd.c
../base/glibc-compatibility/musl/eventfd.c:6:5: error: conflicting types for 'eventfd'; have 'int(unsigned int,  int)'
    6 | int eventfd(unsigned int count, int flags)
      |     ^~~~~~~
In file included from ../base/glibc-compatibility/musl/eventfd.c:1:
/usr/include/sys/eventfd.h:34:12: note: previous declaration of 'eventfd' with type 'int(int,  int)'
   34 | extern int eventfd (int __count, int __flags) __THROW;
      |            ^~~~~~~
cc1: note: unrecognized command-line option '-Wno-builtin-requires-header' may have been intended to silence earlier diagnostics
cc1: note: unrecognized command-line option '-Wno-unused-command-line-argument' may have been intended to silence earlier diagnostics
[34/10091] Building CXX object contrib/libunwind-cmake/CMakeFiles/unwind.dir/__/libunwind/src/libunwind.cpp.o
ninja: build stopped: subcommand failed.
```

解决：修改文件base/glibc-compatibility/musl/eventfd.c，将函数`int eventfd(unsigned int count, int flags)`第1个参数的的类型由`unsigned int`改成`int`。（https://github.com/ClickHouse/ClickHouse/issues/8027 ）
```
int eventfd(/*unsigned*/ int count, int flags)
{
    int r = __syscall(SYS_eventfd2, count, flags);
#ifdef SYS_eventfd
    if (r==-ENOSYS && !flags) r = __syscall(SYS_eventfd, count);
#endif
    return __syscall_ret(r);
}
```

* 编译过程中可能会报错：
```
FAILED: contrib/jemalloc-cmake/CMakeFiles/jemalloc.dir/__/jemalloc/src/pages.c.o 
/usr/bin/gcc -DJEMALLOC_NO_PRIVATE_NAMESPACE -DJEMALLOC_PROF=1 -DJEMALLOC_PROF_LIBUNWIND=1 -DSTD_EXCEPTION_HAS_STACK_TRACE=1 -I../contrib/jemalloc/include -I../base/glibc-compatibility/memcpy -isystem ../contrib/jemalloc-cmake/include -isystem contrib/jemalloc-cmake/include_linux_x86_64/jemalloc/internal -isystem ../contrib/libcxx/include -isystem ../contrib/libcxxabi/include -isystem ../contrib/libunwind/include -fdiagnostics-color=always -include /root/clickhouse/ClickHouse/base/glibc-compatibility/glibc-compat-2.32.h  -pipe -mssse3 -msse4.1 -msse4.2 -mpclmul -mpopcnt -fasynchronous-unwind-tables -ffile-prefix-map=/root/clickhouse/ClickHouse=. -falign-functions=32  -Wall  -Werror -w -O2 -g -DNDEBUG -O3  -fno-pie   -D OS_LINUX -Wno-redundant-decls -D_GNU_SOURCE -std=gnu11 -MD -MT contrib/jemalloc-cmake/CMakeFiles/jemalloc.dir/__/jemalloc/src/pages.c.o -MF contrib/jemalloc-cmake/CMakeFiles/jemalloc.dir/__/jemalloc/src/pages.c.o.d -o contrib/jemalloc-cmake/CMakeFiles/jemalloc.dir/__/jemalloc/src/pages.c.o -c ../contrib/jemalloc/src/pages.c
../contrib/jemalloc/src/pages.c: In function 'pages_purge_lazy':
../contrib/jemalloc/src/pages.c:362:13: error: 'JEMALLOC_MADV_FREE' undeclared (first use in this function)
  362 |             JEMALLOC_MADV_FREE
      |             ^~~~~~~~~~~~~~~~~~
../contrib/jemalloc/src/pages.c:362:13: note: each undeclared identifier is reported only once for each function it appears in
[825/9622] Building CXX object contrib/hyperscan-cmake/CMakeFiles/hyperscan.dir/__/hyperscan/src/rose/rose_build_merge.cpp.o
ninja: build stopped: subcommand failed.
```
解决：修改文件contrib/jemalloc-cmake/include_linux_x86_64/jemalloc/internal/jemalloc_internal_defs.h.in，将宏定义`#define JEMALLOC_PURGE_MADVISE_FREE`注释掉。（https://github.com/ClickHouse/ClickHouse/issues/8027 ）

* 编译过程中可能会报错：
```
FAILED: src/libdbms.a 
: && /usr/local/cmake3-20/bin/cmake -E rm -f src/libdbms.a && /usr/bin/ar qc src/libdbms.a  src/CMakeFiles/dbms.dir/AggregateFunctions/AggregateFunctionCount.cpp.o
                                         ......
src/CMakeFiles/dbms.dir/Coordination/KeeperSnapshotManager.cpp.o src/CMakeFiles/dbms.dir/Coordination/KeeperStateMachine.cpp.o src/CMakeFiles/dbms.dir/Coordination/KeeperStateManager.cpp.o src/CMakeFiles/dbms.dir/Coordination/KeeperStorage.cpp.o src/CMakeFiles/dbms.dir/Coordination/SessionExpiryQueue.cpp.o src/CMakeFiles/dbms.dir/Coordination/SummingStateMachine.cpp.o src/CMakeFiles/dbms.dir/Coordination/WriteBufferFromNuraftBuffer.cpp.o src/CMakeFiles/dbms.dir/Coordination/ZooKeeperDataReader.cpp.o && /usr/bin/ranlib src/libdbms.a && :
/usr/bin/ar: src/libdbms.a: File truncated
ninja: build stopped: subcommand failed.
```
原因：编译生成的libdbms.a文件太大了（超过2GB或4GB），链接的时候不支持大文件。

解决：修改文件CMakeLists.txt，将build类型由`RelWithDebInfo`修改为`Release`。（https://github.com/ClickHouse/ClickHouse/issues/23298 ）
```
if (NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "None")
    set (CMAKE_BUILD_TYPE "Release")                                                                                                                                                                        
    message (STATUS "CMAKE_BUILD_TYPE is not set, set to default = ${CMAKE_BUILD_TYPE}")
endif ()
message (STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
```


### 8. ClickHouse编译生成包测试

编译ClickHouse源码，生成的可执行文件保存在`build/programs/`目录下。

执行以下命令启动server进程：
```
[root@b34940c19b29 build]# cd programs
[root@b34940c19b29 programs]# ./clickhouse server                                                     # 启动server进程
Processing configuration file 'config.xml'.
There is no file 'config.xml', will use embedded config.
Logging trace to console
2021.12.15 13:04:14.231805 [ 148084 ] {} <Information> SentryWriter: Sending crash reports is disabled
2021.12.15 13:04:14.250769 [ 148084 ] {} <Trace> Pipe: Pipe capacity is 1.00 MiB
2021.12.15 13:04:14.401474 [ 148084 ] {} <Information> : Starting ClickHouse 21.11.7.1 with revision 54456, no build id, PID 148084
2021.12.15 13:04:14.401707 [ 148084 ] {} <Information> Application: starting up
2021.12.15 13:04:14.401758 [ 148084 ] {} <Information> Application: OS name: Linux, version: 3.10.0-514.el7.x86_64, architecture: x86_64
2021.12.15 13:04:14.901638 [ 148084 ] {} <Warning> Application: Calculated checksum of the binary: 8F5103E962E4C7343A9967E2C225EB73. There is no information about the reference checksum.
2021.12.15 13:04:14.901847 [ 148084 ] {} <Trace> Application: Will do mlock to prevent executable memory from being paged out. It may take a few seconds.
2021.12.15 13:04:14.968085 [ 148084 ] {} <Trace> Application: The memory map of clickhouse executable has been mlock'ed, total 363.00 MiB
2021.12.15 13:04:14.968590 [ 148084 ] {} <Debug> Application: rlimit on number of file descriptors is 1048576
2021.12.15 13:04:14.968621 [ 148084 ] {} <Debug> Application: Initializing DateLUT.
2021.12.15 13:04:14.968636 [ 148084 ] {} <Trace> Application: Initialized DateLUT with time zone 'UTC'.
2021.12.15 13:04:14.968662 [ 148084 ] {} <Debug> Application: Setting up ./tmp/ to store temporary data in it
2021.12.15 13:04:14.968994 [ 148084 ] {} <Debug> Application: Initiailizing interserver credentials.
2021.12.15 13:04:14.970381 [ 148084 ] {} <Debug> ConfigReloader: Loading config 'config.xml'
Processing configuration file 'config.xml'.
There is no file 'config.xml', will use embedded config.
Saved preprocessed configuration to './preprocessed_configs/config.xml'.
2021.12.15 13:04:14.970844 [ 148084 ] {} <Debug> ConfigReloader: Loaded config 'config.xml', performing update on configuration
2021.12.15 13:04:14.971192 [ 148084 ] {} <Information> Application: Setting max_server_memory_usage was set to 113.08 GiB (125.64 GiB ava
```

执行以下命令启动client进程：
```
[root@b34940c19b29 build]# cd programs
[root@b34940c19b29 programs]# ./clickhouse client                                        # 启动server进程
ClickHouse client version 21.11.7.1.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 21.11.7 revision 54450.

b34940c19b29 :) 
b34940c19b29 :) show databases;                                                                                # 查看集群中的默认的库信息

SHOW DATABASES

Query id: bb3d3542-edea-488f-918c-ed864a3519cb

┌─name───────────────┐
│ INFORMATION_SCHEMA │
│ default            │
│ information_schema │
│ system             │
└────────────────────┘


4 rows in set. Elapsed: 0.002 sec. 

b34940c19b29 :) use information_schema;                                                              # 切换到information_schema库

USE information_schema

Query id: 311f46b2-5063-473f-8908-b88a47828e7b

Ok.

0 rows in set. Elapsed: 0.001 sec. 

b34940c19b29 :) show tables;                                                                                          # 查看库下的所有表信息

SHOW TABLES

Query id: b79362b6-0dd8-41ed-bbe0-5b2d3fd33427

┌─name─────┐
│ columns  │
│ schemata │
│ tables   │
│ views    │
└──────────┘

4 rows in set. Elapsed: 0.003 sec. 

b34940c19b29 :)
```

注：将安装好的环境打包成docker镜像供部署使用。
```
[weizuo@computer ~]$ docker commit -a="weizuo" -m="docker image for clickhouse at version v21.11.7.9-stable" b34940c19b29 cr.d.xxxxxx.net/clickhouse/hello-world:ck-v21.11.7.9-stable
[weizuo@computer ~]$ docker images
REPOSITORY                                              TAG                                 IMAGE ID            CREATED             SIZE
cr.d.xxxxxx.net/clickhouse/hello-world                       ck-v21.11.7.9-stable                23cda1419dc2        4 days ago          9.309 GB
```