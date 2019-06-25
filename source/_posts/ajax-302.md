---
title: 利用Ajax 302 重定向到登录的骚操作
date: 2019-4-19 18:10:54
tags: 
- Ajax
- net-export
---
昨天同事给我演示了一把用`Ajax` **302** 来重定向到登录界面的实现方式。
<!--more-->

### 过程

流程大致是服务端检测到无权限，返回一个302，在location上写了鉴权的地址。随即，浏览器自动向这个地址发起了请求。￼并得到了状态码401；
![](/post-images/ajax-302-1.png)
![](/post-images/ajax-302-2.png)
得到了状态码后，JS执行转跳到相应的登录页面，就开启了登录流程。登录成功后，则转跳回先前访问的页面。
![](/post-images/ajax-302-3.png)
整个过程还是十分清晰的。

### 想法
看到这一过程的第一反应是，Ajax先来一次重定向意义不大吧。检测到未授权直接返回错误码401，前端就可以转跳了。先返回302再检测401再转跳登录，是不是多此一举了呢？难道这一过程里，后端还做了一些其他动作？

### 找不到的`set cookie`
本来就此打住，谁知道我点着点着，猛然发现，用户信息的cookie是什么时候种下的呢？理论上转去登录时应该就设下了，可是找了`getByLogin`和`unauth`接口都没看到相关内容，而浏览器直接访问`unauth`接口，是可以看到set cookie的。
![](/post-images/ajax-302-4.png)
看到了`Request Headers`中有一个`Provisional headers are shown`的提示，就想着是不是能通过`net-export`分析出来。我们重新执行了一次登录流程，这次，我们可以明确的看到，在`unauth`请求401的时候，设置了含有session信息的cookie。同时，还有一条500的请求，也设置了不同Path的cookie，但是同样没有直接显示出来。
![](/post-images/ajax-302-5.png)
![](/post-images/ajax-302-7.png)
￼
### Network提示Provisional headers are shown
￼搜索了下为什么会出现这种情况。[参考](https://www.cnblogs.com/pqjwyn/p/10042492.html)

>1. 跨域，请求被浏览器拦截
2. 请求被浏览器插件拦截
3. 服务器出错或者超时，没有真正的返回
4. 强缓存from disk cache或者from memory cache，此时也不会显示
￼
### initiator为Other
比如直接在地址栏输入的时候
![](/post-images/ajax-302-6.png)

### 我们可以发现
* Ajax也是可以做302重定向的。并且Ajax的重定向请求，浏览器发起的仍然是一个Ajax。
* 要时刻注意开启跨域，毕竟这是Ajax呀。
