---
layout: post
title: 常用Docker命令
categories: 学习笔记
description: 常用Docker命令。
keywords: Docker
---

## 常用Docker命令

##### docker pull

从远程仓库拉取docker镜像（image）到本地。
```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

注：Docker镜像仓库地址的格式一般是“<域名/IP>[:端口号]”，默认地址是Docker Hub(docker.io)。仓库名是两段式名称，即“<用户名>/<软件名>”，对于Docker Hub，如果不给出用户名，则默认为library，也就是官方镜像。

##### docker run

以镜像（image）为基础启动并运行容器（container）。
```
docker run -it <镜像名称> /bin/bash
```

以镜像为基础启动并运行容器，同时将容器目录与本机目录建立关联。
```
docker run -it --cap-add SYS_PTRACE --privileged=true --network bridge --ip <IP地址> -v <本机目录>:<容器目录> --name=<容器名称> <镜像ID> /bin/bash
```


* `-it`是两个参数，一个是`-i`（交互式操作），一个是`-t`（终端），可以进入bash执行一些命令并查看返回结果，因此我们需要交互式终端。

* `-v`表示将某个容器目录挂载到本机的某个目录。一般情况下，`<容器目录>`不要设为`/root`，否则，容器启动之后终端提示符会变成`bash-4.2$`，而不是期望的`[root@1a150ef6c859 ~]#`。因为终端提示符`[root@1a150ef6c859 ~]#`依赖于`/root`目录下的一些配置文件（.bash_profile、.bashrc），如果将容器中的`/root`目录关联本机中的某个空白目录，会导致`/root`目录也是空白目录，所以会出现终端提示符为变成`bash-4.2$`的情况。

* `--name=`可以为容器指定名称。

* `--cap-add=SYS_PTRACE`表示开启ptrace（JDK 工具依赖于 Linux 的 PTRACE_ATTACH，而是 Docker 自 1.10 在默认的 seccomp 配置文件中禁用了 ptrace。）

* `--privileged=true` 指定容器为特权容器，特权容器拥有所有的capabilities。

* `--rm=true`表示容器退出后随之将其删除；默认情况下，退出的容器并不会立即删除，除非手动 docker rm。

* `--network`指定容器的网络类型。

* `--ip`指定容器的IP地址，设置的IP地址须在网络类型的网段内。

##### docker ps 

列出所有正在运行的容器。
```
docker ps 
```

列出所有容器，包括已停止的容器。
```
docker ps -a
```

##### docker start 

启动容器。
```
docker start <CONTAINER ID|NAME>
```

##### docker exec

执行容器（执行容器之前需要先启动容器）。
```
docker exec -i -t <CONTAINER ID|NAME> /bin/bash
```

执行`CTRL+P+Q`会让容器保持后台执行，并不会退出。通过`docker exec`会再次进入bash交互界面。

##### docker stop 

停止容器。
```
docker stop <CONTAINER ID|NAME>
```

##### docker rm 

删除容器（删除容器之前需要先停止容器）。
```
docker rm <CONTAINER ID|NAME> <CONTAINER ID|NAME> ...
```

一次删除所有所有停止的容器。
```
docker rm $(docker ps -a -q) 
```

##### docker images

列出本机中的所有docker镜像。
```
docker images
```

每一个docker镜像对应的属性依次为：REPOSITORY（镜像的仓库源）、TAG（镜像的标签）、IMAGE ID（镜像ID）、CREATED（镜像创建时间）以及SIZE（镜像大小）。同一仓库源可以有多个 TAG，代表这个仓库源镜像的不同个版本。

##### docker rmi

删除镜像（删除镜像之前需要先删除当前镜像创建的容器）。
```
docker rmi <镜像ID> <镜像ID> ...
```

##### docker commit

根据已经存在的容器创建镜像（创建镜像时必须先通过exit命令退出容器）。
```
docker commit -m="<描述信息>" -a="<作者名称>" <容器ID> <镜像仓库名>:<镜像标签>
```
例如：
```
docker commit -a="weizuo" -m="update brpc lib" d39f94dc563f cr.d.xxxxxx.net/doris/doris:1.2-doris-1.0
```

##### docker tag

重命名镜像仓库和镜像标签。
```
docker tag <镜像仓库名>:<镜像标签>  <新的镜像仓库名>:<新的镜像标签>
```
例如：
```
docker tag busybox:latest cr.d.xxxxxx.net/weizuo/busybox:latest
```

##### docker push

将本地镜像push到镜像的远程仓库。
```
docker push <镜像仓库名>:<镜像标签>
```
例如：
```
docker push cr.d.xxxxxx.net/doris/doris:1.2-doris-1.0
```

##### docker history

查看指定镜像的创建历史。
```
docker history <镜像ID>
```

##### docker build

根据Dockerfile文件构建镜像。
```
docker build -t <镜像仓库名>:<镜像标签> .
```

例如：Dockerfile内容如下：
```
FROM centos:7 AS builder

# install dependencies
RUN yum makecache && yum -y update && yum -y groupinstall 'Development Tools' && \
    yum install -y byacc automake java-11-openjdk-devel java-1.8.0-openjdk-devel libtool bison binutils-devel zip \
    unzip ncurses-devel curl git wget python2 glibc-static java-1.8.0-openjdk-devel \
    libstdc++-static which psl libpsl-devel centos-release-scl && \
    yum install -y devtoolset-10 devtoolset-10-gcc devtoolset-10-libubsan-devel devtoolset-10-liblsan-devel \
    devtoolset-10-libasan-devel

# build cmake
ARG CMAKE_VERSION=3.19.8
ARG CMAKE_BASE_URL=https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}
RUN wget ${CMAKE_BASE_URL}/cmake-${CMAKE_VERSION}-Linux-x86_64.sh -q -O /tmp/cmake-install.sh && \
    chmod u+x /tmp/cmake-install.sh && \
    /tmp/cmake-install.sh --skip-license --prefix=/usr --exclude-subdir && \
    rm /tmp/cmake-install.sh

# build  ninja
ARG NINJA_VER=1.10.2
ARG NINJA_BASE_URL=https://github.com/ninja-build/ninja/releases/download/v${NINJA_VER}
RUN wget -q ${NINJA_BASE_URL}/ninja-linux.zip -O /tmp/ninja-linux.zip && \
    unzip /tmp/ninja-linux.zip -d /usr/bin/ && \
    rm /tmp/ninja-linux.zip

# install maven 3.6.3
ARG MAVEN_VERSION=3.6.3
ARG MAVEN_BASE_URL=https://downloads.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries
RUN mkdir -p /usr/share/maven /usr/share/maven/ref && \
    wget -q -O /tmp/apache-maven.tar.gz ${MAVEN_BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz && \
    tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 && \
    rm -f /tmp/apache-maven.tar.gz && \
    ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

# build flex
ARG FLEX_VERSION=2.6.4
RUN wget https://github.com/westes/flex/releases/download/v$FLEX_VERSION/flex-$FLEX_VERSION.tar.gz \
    -q -O /tmp/flex-$FLEX_VERSION.tar.gz \
    && cd /tmp/ \
    && tar -xf flex-$FLEX_VERSION.tar.gz \
    && cd flex-$FLEX_VERSION \
    && ./configure --enable-shared=NO \
    && make \
    && make install \
    && rm /tmp/flex-$FLEX_VERSION.tar.gz \
    && rm -rf /tmp/flex-$FLEX_VERSION

# install nodejs
ARG NODEJS_VERSION=14.16.0
RUN wget https://nodejs.org/dist/v$NODEJS_VERSION/node-v$NODEJS_VERSION-linux-x64.tar.gz \
    -q -O /tmp/node-v$NODEJS_VERSION-linux-x64.tar.gz \
    && cd /tmp/ && tar -xf node-v$NODEJS_VERSION-linux-x64.tar.gz \
    && cp -r node-v$NODEJS_VERSION-linux-x64/* /usr/local/ \
    && rm /tmp/node-v$NODEJS_VERSION-linux-x64.tar.gz && rm -rf node-v$NODEJS_VERSION-linux-x64


ENV BASH_ENV=/opt/rh/devtoolset-10/enable  \
    ENV=/opt/rh/devtoolset-10/enable  \
    PROMPT_COMMAND=". /opt/rh/devtoolset-10/enable"

# there is a repo which is included all of thirdparty
ENV JAVA_HOME="/usr/lib/jvm/java-11"

RUN alternatives --set java java-11-openjdk.x86_64 && alternatives --set javac java-11-openjdk.x86_64

FROM scratch

COPY --from=builder / /
ENV JAVA_HOME="/usr/lib/jvm/java-11" \
    MAVEN_HOME="/usr/share/maven" \
    BASH_ENV="/opt/rh/devtoolset-10/enable" \
    ENV="/opt/rh/devtoolset-10/enable"  \
    PROMPT_COMMAND=". /opt/rh/devtoolset-10/enable"
WORKDIR /root

CMD ["/bin/bash"]
```

在Dockerfile文件所在的目录下执行 `sudo docker build -t doris/doris:doris-1.0  --no-cache .` 命令构建docker镜像，设置仓库为`doris/doris`，tag为`doris-1.0`，`--no-cache`表示构建的时候不使用缓存。

```
weizuo@pc:~/Dockerfile$ ll
总用量 20
drwxrwxr-x  2 mi mi  4096 8月  16 10:32 ./
drwxr-xr-x 12 mi mi 12288 8月  16 10:31 ../
-rw-rw-r--  1 mi mi  3130 8月  16 10:32 Dockerfile

weizuo@pc:~/Dockerfile$ sudo docker build -t doris/doris:doris-1.0 --no-cache .
Sending build context to Docker daemon   5.12kB
Step 1/23 : FROM centos:7 AS builder
 ---> 8652b9f0cb4c
Step 2/23 : RUN yum makecache && yum -y update && yum -y groupinstall 'Development Tools' &&     yum install -y byacc automake java-11-openjdk-devel java-1.8.0-openjdk-devel libtool bison binutils-devel zip     unzip ncurses-devel curl git wget python2 glibc-static java-1.8.0-openjdk-devel     libstdc++-static which psl libpsl-devel centos-release-scl &&     yum install -y devtoolset-10 devtoolset-10-gcc devtoolset-10-libubsan-devel devtoolset-10-liblsan-devel     devtoolset-10-libasan-devel
 ---> Running in e17e41e6ee46

 ......

```

docker镜像构建完成之后，执行`sudo docker images`命令查看构件好的docker镜像。
```
weizuo@pc:~/Dockerfile$ sudo docker images
REPOSITORY                    TAG                 IMAGE ID       CREATED          SIZE
doris/doris                   doris-1.0           d581dab6861e   6 minutes ago    1.84GB
```

执行`sudo docker history <镜像ID>`命令查看docker镜像的创建历史。
```
weizuo@pc:~/Dockerfile$ sudo docker history d581dab6861e
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
d581dab6861e   7 minutes ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B        
2670e143548e   7 minutes ago   /bin/sh -c #(nop) WORKDIR /root                 0B        
1015d3b4a6ad   7 minutes ago   /bin/sh -c #(nop)  ENV JAVA_HOME=/usr/lib/jv…   0B        
876f4ac89c8b   7 minutes ago   /bin/sh -c #(nop) COPY dir:1d2de43d239d90dd6…   1.84GB  
```

##### docker network

在Docker中，默认情况下容器与容器、容器与外部宿主机的网络是隔离开来的。当安装Docker的时候，docker会创建一个桥接器docker0，通过它才能让容器与容器之间、与宿主机之间通信。

Docker安装的时候默认会创建三个不同的网络，可以通过`docker network ls`命令查看这些网络：
```
weizuo@pc:~$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
f65bddc829ad        bridge              bridge              local
887f3f66f5dc        host                host                local
7d7c2584672c        none                null                local
```

* none模式
使用none网络模式创建容器时，不会为容器创建任何的网络环境，容器内部就只能使用loopback网络设备，不会再有其他的网络资源。

* host模式
使用none网络模式创建容器时，容器会使用宿主机的网络，容器与宿主机的网络并没有隔离。使用这种网络类型的好处就是网络性能很好，基本上跟宿主机的网络一样，它很大的弊端就是不安全。你可以在容器中更改宿主机的网络，如果你的程序是用root用户运行的，有可能会通过Docker容器来控制宿主机的网络。当我们在容器中执行类似ifconfig命令查看网络环境时，看到的都是宿主机上的信息。

* bridge模式
桥接网络是默认的网络类型，桥接的网络名为docker0。当我们启动一个容器的时候，每个容器会有它自己的虚拟网络接口连接到docker0，并获得一个IP地址。上面的 Containers 项里有所有连接到这个网络的容器，可以看到这个容器的ip地址。

可以选择空闲的网段，通过以下命令创建新的网络：
```
docker network create --subnet=网段/子网掩码位数 网络名称
```
例如：
```
weizuo@pc:~$ sudo docker network create --subnet=172.18.0.0/24 network-wz
```
创建容器时为容器赋予固定的IP地址：
```
weizuo@pc:~$ sudo docker run -it --cap-add SYS_PTRACE --privileged=true --network network-wz --ip 172.18.0.101 --name=clickhouse-node1 -v /home/weizuo/ClickHouse/docker:/root/clickhouse 23cda1419dc2 /bin/bash
```
