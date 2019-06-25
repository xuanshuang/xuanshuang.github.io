---
title: Session的炒冷饭
date: 2019-06-23 15:51:14
tags: 
- Session
- cookie
- net-export
---
最近App的同学反应需要鉴权的H5在首次打开的时候，都拿不到用户详情。刷新或关闭H5后再次打开，才恢复正常。经过定位是`Session`的问题。
<!--more-->

## 定位
### 过程
首先，通过`Network`发现没有异常的请求，但是用户信息的接口返回了空值。查看日志后发现用户是成功登陆的，但是登陆状态在查询用户信息时失效了。检查了node服务也发现oAuth一切正常，猜测是`Session`的问题。对比登陆种下的cookie和获取用户信息发送的cookie后，发现它们居然不是同一个。是在哪里发生了改变呢？我们打开`Network`的`Set Cookies`和`Cookies`标记，在Chrome 74里我们发现：

![](/post-images/session-0.jpg)

天啦噜，`Session`的`cookie`的确变化了,但是实在哪里变的呢？
接着我尝试使用net-export导出信息，却发现`cookie`信息都被**145 bytes were stripped**了。。。

![](/post-images/session-1.jpg)

搞不清Chrome的规则，尝试在Chrome 70中抓去，发现多了很多cookie相关的内容:

![](/post-images/session-2.jpg)

我们发现，有一条请求静态资源`hybirdpi.js`的请求，状态码是`302`，并且设置了新的cookie。正是这条新的cookie，导致了`Session`无效。那又是为什么会设置了新的Cookie呢？
我们一开始登陆种下cookie的地方是：
![](/post-images/session-3.png)
新设置的cookie的地方是：
![](/post-images/session-4.png)
猛然发现他们端口号不一样。**而Cookie的设置是不区分端口号的**。原来这是两个独立的服务，分别为两个类型的微应用做认证服务，却设置了相同的`session key`、`Path`，又不能区分端口，就相互覆盖了。
修改好了相关逻辑，提交到线上，又出现了问题。这次是为了抵抗并发压力，线上开启了多实例。而我们的session机制暂时没有支持多实例共享，所以用户信息又乱套了。
![](/post-images/session-5.png)

> 在大型网站中，我们的服务器通常不止一台，可能是几十台甚至几百台之多，甚至多个机房都可能在不同的省份，用户发起的HTTP请求通常要经过像Ngnix之类的负载均衡器之后，再路由到具体的服务器上，由于Session默认存储在单机服务器内存中，因此在分布式环境下同一个用户发送的多次HTTP请求可能会先后落到不同的服务器上，导致后面发起的HTTP请求无法拿到之前的HTTP请求存储在服务器中的Session数据，从而使得Session机制在分布式环境下失效，因此在分布式集群中CSRF Token需要存储在Redis之类的公共存储空间。[参考](https://tech.meituan.com/2018/10/11/fe-security-csrf.html)

## 吐槽
定位的过程简直是吐槽Chrome的过程。出于安全性抑或是其它的考虑，Chrome新版本的devtool中对Cookie的修改做了蛮多限制，比如不能直接在Application中修改`Http-only`的cookie;某些情况下（如我们今天遇到的）省略`Set Cookies`的标记。

## 相关
解决问题后，觉得以后设置`cookie`的时候还是需要谨慎，比如Path不要设为/；cookie的key不同环境要区分等等。顺道复习了一下cookie的知识。
* [cookie不区分端口号](https://stackoverflow.com/questions/1612177/are-http-cookies-port-specific)
> Cookies do not provide isolation by port. If a cookie is readable by a service running on one port, the cookie is also readable by a service running on another port of the same server. If a cookie is writable by a service on one port, the cookie is also writable by a service running on another port of the same server. For this reason, servers SHOULD NOT both run mutually distrusting services on different ports of the same host and use cookies to store security sensitive information.

* [cookie参考教程](http://javascript.ruanyifeng.com/bom/cookie.html#toc4)
