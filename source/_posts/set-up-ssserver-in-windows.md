---
title: window server搭建ssserver
date: 2019-03-13 08:18:01
tags: 
- shadowsocks
---
公司封了网易云音乐，想着有台公网的ECS，拿来做代理吧。ECS用的window server，捣鼓捣鼓吧。

<!--more-->

## 安装过程

搜了下，在window下一般用python来搭建ss服务端的。

### Step 1
安装python 2.7。安装包内置pip，方便下载python的各种lib。在安装过程中，记得勾选`安装环境变量`。
### Step 2
安装shadowsocks服务端。Python会把网上安装的库放到安装目录下/python27/lib/site-packages/shadowsocks。
``` py
pip install shadowsocks // 若pip环境变量无效 则python -m pip install shadowsocks
```
### Step 3
安装openssl，去官网下载最小版本即可。记得勾选复制到windows/system32目录下。pip库中装的shadowsocks是2.8.2版本，和openssl的dll文件有冲突，在安装openssl后需要额外操作：
* 把libcrypto-1_1.dll改为libcrypto.dll
* 到python27/lib/site-packages/shadowsocks/crypto/openssl.py修改libcrypto.EVP_CIPHER_CTX_cleanup为libcrypto.EVP_CIPHER_CTX_reset，共两处

### Step 4
* 写一个配置文件
```
{
  "server":"0.0.0.0",
  "server_port":8888,
  "password":"your_password",
  "timeout":600,
  "method":"aes-256-cfb",
  "fast_open": false
}
```

* 启动
``` js
ssserver -c /config.json // 类unix系统中还有参数-d，可是设置在后台运行，window无效
// 停止ss服务：ssserver -c /etc/shadowsocks.json -d stop
// 启动ss服务：ssserver -c /etc/shadowsocks.json -d start
// 重启ss服务：ssserver -c /etc/shadowsocks.json -d restart
```

### Step 5
切记去阿里云安全组开放端口。

## 过程中了解到的一些东西

* 为shadowsocks开启BBR加速，BBR是Google开源的一套内核加速算法，可以让你搭建的shadowsocks体验飞一般的感觉，浏览速度有个质的提升
* OpenVZ架构VPS无法更新内核
* 127.0.0.1和0.0.0.0的区别
  * 3306端口监听在127.0.0.1，只有本机客户端可以访问，其他服务器无法访问
  * 3306端口如果监听在0.0.0.0上，如果没有端口限制，那么其他服务器则可以连接该服务器的该端口
* Centos named 服务没起 无法域名解析  service named start；无法在服务器使用curl命令访问https域名,原因是nss版本有点旧了，yum -y update nss更新一下，重新curl即可。
* 查看系统整体的负载-top;总体内存占用的查看-free;查看进程：ps -A | grep xxx
* 什么是KCPTUN

## 参考链接
http://blog.51cto.com/14018334/2299803
https://blog.csdn.net/MaoshiYIHAO/article/details/84777683
https://www.flyzy2005.cn/tech/build-shadowsocks-on-vps/
[以后要是在阿里云买香港的，最好把云盾删除](https://www.moerats.com/archives/807/)
[OpenVZ架构 & CentOS 6 x64 搭建BBR](https://blog.csdn.net/qq_43044935/article/details/83964335)
[openvz 开启bbr](https://www.moerats.com/archives/111/)
[openvz 开启bbr](https://www.moerats.com/archives/190/)
[OpenVZ平台魔改BBR一键脚本之Rinetd方式](https://www.moerats.com/archives/504/)
