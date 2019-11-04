---
title: 容器指令备忘
date: 2019-11-04 12:50:15
tags: 
- docker
- docker-compose
- mongo
---
如今容器技术已经十分成熟了，前后端或运维的日常都离不开它。平时老是用公司线上CI/CD平台来完成部署，突然有了一台隔离环境需要徒手敲命令，却发现自己把命令行的操作指令遗忘了七七八八。这里记录一下docker的常用指令，特别是在单台环境部署多个服务时，docker-compose的内容。
<!--more-->

### 基本操作
* 登陆docker hub `docker login`。
* 拉取镜像 `docker pull`。
* 给镜像打tag `docker tag image-id xxxx:yyy`。
* 查看image `docker image ls`。
* 查看container `docker ps`会显示当前正在运行的镜像。ps:有些时候镜像运行完就退出了，可以加上`-a`参数来找到它。
* 杀掉某一正在运行镜像 `docker kill xxx`。xxx可以是镜像的名称、container Id等等。
* 查看镜像的log `docker logs 464fced66de3`。可以帮助定位出问题的镜像。
* 查看本地存储卷 `docker volume ls`。
* 运行镜像 `docker run -p localhost-port:image-expose-port -d your-docker-image`。`-p`绑定端口，`-d`让镜像可以在后台运行，`-e`增加环境变量。这里还有一堆参数，请看`docker run`指令。注意，`your-docker-image`后面还可以增加类似`/bin/bash`的指令，如果镜像是用Dockerfile打出来的话，这句话会覆盖你的`CMD`指令。
* 镜像打包 `docker save 4837420db37e | gzip > 4837420db37e.tgz`。
* 镜像复原 `docker load -i 4837420db37e.tgz`。这两个指令可以方便的迁移我们的镜像。
* 自动清理 `docker [commmand like: volume/container] prune`。`docker rm $(docker ps -aq)`。`docker system prune --volumes`清除所有不相关或不使用的Docker数据。
* 编译镜像 `docker build -f your-Dockerfile`。
* 查看docker的事件。这时，我们需要多开一个shell进入`docker events`中。在另一个shell中，正常执行我们的指令。你会发现，我们在第一个shell中可以实时显示容器的内部事件了。
* PS:各种命令都可以加上`docker xxx --help`来快速查看额外参数的作用。

有时候我们想要运行镜像后，进入此镜像的shell中。这里有三个有用的指令，一个用于镜像启动时，一个用于启动后：
* `docker run -it your-image`适用于运行镜像时。
* `docker attach container-id`用来连接到正在运行中的容器中。
* `docker exec [OPTIONS] container-id COMMAND [ARG...]`用来在运行的镜像中执行相关命令。我们也可以使用`docker exec -it container-id /bin/bash`来进入一个运行的镜像。


### 利用docker-compose部署多个服务
在需要同时部署多个服务的时，我们当然可以每次运行一个镜像来启动每一个服务。但有时候，如果服务直接存在相互的依赖关系，一个服务未启动会使另一个服务挂掉，就需要`docker-compose`出马了。我们这里以一个node服务和一个mongo服务为例，且node服务是依赖mongo服务的。

#### 写一个"compose-file"
我们需要在合适的目录下，建立一个`docker-compose.yml`文件。
``` YAML
version: '3.7'

services:
  mongodb:
    image: mongo:3.4.3
    container_name: mongodb
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: root
      MONGO_INITDB_DATABASE: admin
      MONGO_USERNAME: dbuser
      MONGO_PASSWORD: dbpwd
      MONGO_DATABASE: dbname
    ports:
      - 27017:27017
    volumes:
      - ./mongo-init.sh:/docker-entrypoint-initdb.d/mongo-init.sh:ro
      - mongo-volume:/data/db
  web:
    image: my-web:latest
    container_name: web
    depends_on:
      - "mongodb"
    ports:
      - 3000:3000
      - 6789:6789
    environment:
      PORT: '3000'

volumes:
  mongo-volume:
    external: false
```
最外层并行的三个配置项是：
* `version`指的是使用的compose文件的格式版本。目前使用的是*3.7*。
* `services`需要docker启动的服务。这个对象下的每一项，都是一个会被启动的服务。
* `volumes`需要docker创建的内部文件系统。同理，它的每一项，都将生成一个存储卷。

ports和expose的区别，前者可以被外部访问，expose暴露的只能在容器内访问[参考](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-docker-compose-ports-vs-expose)。


其他子配置项的作用可以在[官网](https://docs.docker.com/compose/)中寻找，这里就不赘述了。下面就本次工程，指出几个关键的配置的作用。

#### 对于mongo
在建立数据库服务时，我们会考虑下面的问题：
1. 初始数据如何导入？
2. 数据如何持久化？

在查询了官方mongo镜像的[Dockerfile](https://github.com/docker-library/mongo/blob/master/3.4/docker-entrypoint.sh#L197)后，我们可以通过一些环境变量来配置数据库。简单来说，如果环境变量`MONGO_INITDB_ROOT_USERNAME`和`MONGO_INITDB_ROOT_PASSWORD`存在，将会创建一个*admin*数据库和一个root用户，用来进行后续管理。同时，在启动镜像前，mongo将会执行`/docker-entrypoint-initdb.d/`下的文件，我们通过这个方式，可以完成数据库的初始化。

对于第二个问题，docker为我们提供了[*volume*](https://docs.docker.com/storage/volumes/)存储卷模式。在上面的配置文件中，我们通过`volumes`指定了要创建的存储卷mongo-volume。然后我们在服务mongodb下，定义了volumes的使用方式：
``` bash
- ./mongo-init.sh:/docker-entrypoint-initdb.d/mongo-init.sh:ro # 这里把我们的初始化脚本复制进执行目录。参数:ro的意思是只读。
- mongo-volume:/data/db # 这里挂载了mongodb存储位置
```
下面是初始化脚本

``` bash
#!/bin/bash
"${mongo[@]}" "$MONGO_INITDB_DATABASE" <<EOF
  var dbname = db.getSiblingDB('dbname');
  var user = "$MONGO_USERNAME";
  var passwd = "$MONGO_PASSWORD";
  dbname.createUser({user: user, pwd: passwd, roles:[{role: "readWrite",db: "dbname"}] });
EOF
```

另外推荐一个mongo可视化连接工具*Robo 3T*来方便操作。
#### 对于node
node服务其实比较简单，通过Dockerfile构建好镜像后，暴露出相应的服务端口即可。在这里，`depends_on`解决了上面提到的依赖问题。此时，docker会先启动被依赖的服务。这里记录一个Dockerfile，当碰到依赖库(imagemin-pngquant)需要主动编译情况下的解决方案：

``` Dockerfile
FROM node:10.16-alpine
LABEL author=my@my.com
WORKDIR /home/my-web
COPY . ./
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
RUN apk update && \ 
  apk add --no-cache git \
    autoconf \
    automake \
    bash \
    g++ \
    libc6-compat \
    libjpeg-turbo-dev \
    libpng-dev \
    make \
    nasm # 安装这么一大堆东西来编译imagemin-pngquant的依赖

RUN npm install  && \
  npm run build

CMD ["npm", "run", "start:prod"]
```

需要指出的是，默认情况下，镜像里的localhost不是你本机的localhost啦。我就是忘记了这一点，在调试的过程中浪费了许多时间。docker-compose在启动的时候，会建立一系列的工作，其中之一便是建立一个[虚拟网络](https://docs.docker.com/compose/networking/)。所以我们在node服务中连接数据库时，数据库的host是你的服务名*mongo*（上面的配置文件中）了。正确的写法应该是:`mongodb://dbuser:dbpwd@mongo:27017/dbname`。当然，这种网络模式也可以改变，可以参考网络的*bridge mode*和*host mode*。

#### 启动/更新/关闭
* `docker-compose up -d`-启动服务。默认将会在当前目录下寻找配置文件*docker-compose.yml*。
* `docker-compose pull web && docker-compose down && docker-compose up -d`-更新镜像并重启，数据不会丢失。
* `docker-compose down`-关闭并销毁。

### 参考
[Docker-YApi](https://github.com/fjc0k/docker-YApi)
[Managing MongoDB on docker with docker-compose](https://medium.com/faun/managing-mongodb-on-docker-with-docker-compose-26bf8a0bbae3)
[how to set docker mongo data volume](https://stackoverflow.com/questions/35400740/how-to-set-docker-mongo-data-volume)


<!-- [https://stackoverflow.com/questions/39348478/initialize-data-on-dockerized-mongo]
[https://stackoverflow.com/questions/39282957/mongorestore-in-a-dockerfile]
[https://stackoverflow.com/questions/38298645/how-should-i-backup-restore-docker-named-volumes]
[https://stackoverflow.com/questions/42912755/how-to-create-a-db-for-mongodb-container-on-start-up/42917632#42917632] -->


### 最后
到了最后，发现这篇就是一个大杂烩，也不知道再看还有没有效果～～