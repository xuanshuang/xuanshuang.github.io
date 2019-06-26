---
title: Docker缓存问题
date: 2019-02-19 10:10:54
tags: 
- Docker
---
最近在使用公司CI/CD工具部署前端镜像时，发现运行的Web都是第一版的内容。通过查看构建日志，发现在打镜像的时候，许多步骤都运用了缓存。
<!--more-->

### 构建日志

``` log
############ Build Image ############
Sending build context to Docker daemon 4.608 kB
Step 1/6 : FROM nginx:1.13-alpine
1.13-alpine: Pulling from library/nginx
Digest: sha256:9d46fd628d54ebe1633ee3cf0fe2acfcc419cfae541c63056530e39cd5620366
Status: Image is up to date for nginx:1.13-alpine
---> ebe2c7c61055
Step 2/6 : MAINTAINER dopware 
---> Using cache
---> 8e9f463cbf07
Step 3/6 : RUN apk update && apk add --no-cache --no-progress tzdata curl bash
---> Using cache
---> c1ad7aaa5ce3
Step 4/6 : RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
---> Using cache
---> 81a315cb9403
Step 5/6 : COPY nginx.conf /etc/nginx/nginx.conf
---> Using cache
---> 58a22127d165
Step 6/6 : RUN mkdir -p /home/legov/web2b && cd /home/legov/web2b && curl -s http://****/legov-web.tar.gz | tar xz
---> Using cache
---> 74e154a59f7a
Successfully built 74e154a59f7a
Login Succeeded
The push refers to a repository [hub.***.**/legovwebapp/legov-web]
30494122bcfb: Preparing
6604a15fc13c: Preparing
...
87deea508850: Waiting
...
6604a15fc13c: Layer already exists
...
cd7100a72410: Layer already exists
90c4db1d5ef5: Layer already exists
latest: digest: sha256:38ff68d088c1424c644997df77a2ec468192ab8075f10ac23ec35dc0b1d04f20 size: 1989

############ Build  finished ############
```
## 分析问题
气势汹汹杀向工具的开发同学，问曰：可以使用`docker build --no-cache`命令吗？答曰：还不支持，下个版本给你加上！
只能寻求它法，然后该同学给我普及了一波缓存知识以助我迈过这个坑。

### 什么是Docker命令的缓存
Docker对每一条命令构建都会产生一个镜像缓存，如果下次构建命中了此次缓存，则就不进行新的构建而是使用缓存镜像。
如上面日志中，存在`--->Using Cache`的地方，就命中并使用了缓存。
Docker缓存策略
* 从已经在缓存中的父镜像开始，将下一个指令与从该基本镜像导出的所有子镜像进行比较，以查看其中一个是否使用完全相同的指令构建。如果没有，则缓存无效。
* 在大多数情况下，只需将Dockerfile中的指令与其中一个子镜像进行比较即可（通过比较是否与上一次执行的指令一致）。但是，某些指令需要一些额外的检查。对于ADD和COPY指令，将检查镜像中文件的内容，并为每个文件计算校验和。在这些校验和中不考虑文件的最后修改和最后访问的时间。在缓存查找期间，将校验和与现有映像中的校验和进行比较。如果文件（如内容和元数据）中有任何变化，则缓存无效。
* 除了ADD和COPY命令之外，缓存检查将不会查看容器中的文件来确定缓存匹配。例如，当处理RUN apt-get -y update命令时，不会检查在容器中更新的文件以确定是否存在高速缓存命中，它将只会检查命令字符串是否与之前的一致来判断是否匹配。一旦某一层的缓存无效，所有后续的Dockerfile命令将生成新的镜像，并且高速缓存将不被使用。

### 为什么我们会命中缓存呢
通过检查Dockerfile，我们可以发现命令5是这么写的:

* `RUN mkdir ***** && curl *** | tar ***` 下载构建产物

我们更新Web资源，主要通过curl去获取构建产物。这一步写在了最后的RUN命令中，由于之前已经运行过相同的命令了，所以接下来版本的构建中直接命中并使用了缓存。

### 怎么禁止使用缓存呢
* 在制作镜像的时候，使用`docker build --no-cache`。但是此处CI/CD系统功能暂不支持。
* 上面分析了我们的缓存是因为使用了RUN命令中使用curl来获取构建产物，而这整条RUN被缓存了。我们使用ADD命令，把最后一行改为
  ```
  ADD http://archive.***.**/legovwebapp/legov-web.tar.gz  /home/legov/web2b/
  RUN cd /home/legov/web2b && tar -zxvf legov-web.tar.gz
  ```

#### COPY和ADD缓存策略

判断`ADD`命令或者`COPY`命令后紧接的文件是否发生变化，是否延用cache的重要依据。Docker采取的策略是：获取Dockerfile下内容（包括文件的部分inode信息），计算出一个唯一的hash值，若hash值未发生变化，则可以认为文件内容没有发生变化，可以使用 cache 机制；反之亦然。例如，对于命令`ADD run.sh /`，若当前目录下的`run.sh`发生了变化，原则上不应该再使用cache，从而将直接导致镜像层文件系统内容的更新。

## Docker相关指令
* WORKDIR-指令用于指定容器的一个目录,容器启动时执行的命令会在该目录下执行,相当于设置了容器的工作目录。
* RUN-RUN命令是创建Docker镜像（image）的步骤，RUN命令对Docker容器（ container）造成的改变是会被反映到创建的Docker镜像上的。一个Dockerfile中可以有许多个RUN命令。
* COPY-`COPY <src> <dest>`。
* ADD-ADD指令的功能是将主机构建环境（上下文）目录中的文件和目录、以及一个URL标记的文件拷贝到镜像中。认定用不用缓存，文件的构建时间不一样，也被算在不同的指标中。其格式是：`ADD 源路径 目标路径`。例如，把当前config目录下所有文件拷贝到/config/目录下：`ADD config/ /config/`。该指令还有解压等强大功能，建议阅读文档。
* CMD-CMD命令是当Docker镜像被启动后Docker容器将会默认执行的命令。一个Dockerfile中只能有一个CMD命令。通过执行docker run $image $other_command启动镜像可以重载CMD命令。

## 本地运行docker镜像
假设做好的镜像叫docker-dtd。可以用`docker image ls`查看。
``` bash
# -v 前面本机目录:映射到docker里面的
# 创建一个守护态的Docker容器，然后使用docker attach命令进入该容器
docker run -itd -v ~/Desktop/work/reta-start-kit:/home/project -p 8080:8080 D-dtd /bin/bash
docker ps
CONTAINER ID  IMAGE    COMMAND      CREATED         STATUS          PORTS             
36d8a036bdf6  D-dtd    "/bin/bash"  16 seconds ago  Up 15 seconds   0.0.0.0:8080->8080/tcp
# 第一种进入镜像bash的方法
docker attach 36d8a036bdf6
# 第二种进入镜像bash的方法
docker exec -it --user root 36d8a036bdf6 /bin/bash // 以root身份运行
# 退出
exit
```

## 参考
[Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
[docker build 的 cache 机制](http://guide.daocloud.io/dcs/docker-build-cache-9153988.html)

