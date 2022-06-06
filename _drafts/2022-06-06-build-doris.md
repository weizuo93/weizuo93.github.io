---
layout: post
title: Apache Doris 的编译 
categories: Doris编译
description: 
keywords: 
---

### 安装依赖包
```
sudo apt install python2.7 build-essential g++ gcc mysql-client-core-8.0 libssl-dev byacc flex automake libtool ccache autoconf autopoint maven bison flex binutils-devel perf util-linux ncurses-devel gettext-devel binutils-dev libiberty-dev nodejs nodejs-legacy npm
```

### 安装gcc
```
mi@mi:~$ gcc -v
使用内建 specs。
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-linux-gnu/11/libexec/gcc/x86_64-pc-linux-gnu/11.1.0/lto-wrapper
目标：x86_64-pc-linux-gnu
配置为：../configure --prefix=/usr/lib/gcc/x86_64-linux-gnu/11 --enable-bootstrap --enable-checking=release --enable-languages=c,c++ --disable-multilib
线程模型：posix
Supported LTO compression algorithms: zlib
gcc 版本 11.1.0 (GCC) 
```

### 安装cmake
```
mi@mi:~$ cmake --version
cmake version 3.22.1

CMake suite maintained and supported by Kitware (kitware.com/cmake).

```