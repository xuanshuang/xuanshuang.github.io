---
title: 复习下OPTIONS
date: 2019-04-06 20:19:18
tags: 
- 跨域
- OPTIONS
- 429
---
最近要做一个服务限流的功能，前后端约定通过状态码`429`来区分。后端同学采用了拦截器的方案，在调试过程中遇到了跨域的问题，借此复习下`OPTIONS`。
<!--more-->

### 问题
实际调试中发现：
1. 正常请求可以通过跨域
![](/post-images/options-1.png)
2. `429`的请求通过了`OPTIONS`检查，真实请求发生跨域报错。
![](/post-images/options-2.png)

￼原来后端同学忘记限流拦截器中返回`CORS header`了。
![](/post-images/options-3.png)
￼
### OPTIONS
解决了问题后，发现CORS的知识有一些忘记了。为什么通过`OPTIONS`请求后，仍然需要`CORS`的header呢？[参考](http://www.ruanyifeng.com/blog/2016/04/cors.html)
* `OPTIONS`是预检查请求
* 通过后发起正常请求，也需要Access-Control-Allow-Origin

>非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错。
"预检"请求用的请求方法是OPTIONS，表示这个请求是用来询问的。头信息里面，关键字段是`Origin`，表示请求来自哪个源。
除了Origin字段，"预检"请求的头信息包括两个特殊字段。
（1）Access-Control-Request-Method
该字段是必须的，用来列出浏览器的CORS请求会用到哪些HTTP方法，上例是PUT。
（2）Access-Control-Request-Headers
该字段是一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段。

>服务器收到"预检"请求以后，检查了Origin、Access-Control-Request-Method和Access-Control-Request-Headers字段以后，确认允许跨源请求，就可以做出回应。

>一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个Origin头信息字段。服务器的回应，也都会有一个`Access-Control-Allow-Origin`头信息字段(**这里就是上面发送错误的原因**)。如果服务器否定了"预检"请求，会返回一个正常的HTTP回应，但是没有任何CORS相关的头信息字段。这时，浏览器就会认定，服务器不同意预检请求，因此触发一个错误，被XMLHttpRequest对象的onerror回调函数捕获。

### 其他限流方案
[nginx限流](https://www.cnblogs.com/biglittleant/p/8979915.html)
