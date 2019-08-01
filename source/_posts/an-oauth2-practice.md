---
title: 一次对接oauth2实践
date: 2019-08-01 11:11:11
tags: 
- OAuth2
- backend
---
最近使用了YApi来管理接口相关的内容，但是每个人都要手动注册。大家觉得麻烦，就需要对接一下Auth。
<!--more-->

### OAuth2
准备采用OAuth2.0来进行对接。其中使用授权码来进行接入的流程如下：
* 从我们的应用请求认证服务器获取code。这时认证服务器会转跳到他们的用户登陆页面，提醒你是否确认登陆并授权给我们的应用。
* 认证服务器通过了认证，回调到我们在上一步提供的回调地址。认证服务器会在回调地址上拼接code和state参数，给我们去换取token。
* 我们的服务器拿到code和state后，向认证服务器请求获得此次授权的用户token。
* 拿到token后，我们就可以向资源服务器（可能就是认证服务器）请求我们需要的资源了。在这里，就是用户信息。

### 流程
那么，基于这个流程，我们需要认证服务器提供三个接口：
1. 获取code（登陆）接口。`GET TheirHost/api/base/oauth/authorizationCode`。它有如下几个query参数：
| 参数 | 说明 |
| :-- | :---- |
| response_type=code | 标明这是采用授权码方式 |
| client_id=client_id| 我们应用在认证服务器注册的标识 |
| redirect_uri=redirect_uri | 回调Url。授权成功后，回调到我们自己的后端 |
| scope | 需要授权的范围 |
| state | 可用来CSRF防御 |

2. 获取token接口。`POST TheirHost/api/base/oauth/token`。它有如下query参数：
| 参数 | 说明 |
| :-- | :---- |
| grant_type=authorization_code | 标明这是采用授权码方式来换取code |
| client_id=client_id| 我们应用在认证服务器注册的标识 |
| redirect_uri=redirect_uri | 回调Url。必须与上面转跳时给的一致，和code一起被用来确认身份。 |
| code=code | 上一步中返回的code |
| client_secret=client_secret | 我们应用在认证服务器上注册的密钥 |

它的标准返回值如下：
``` json
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create delete"
}
```
3. 根据token获取用户信息。 `POST TheirHost/api/base/oauth/user?access_token=xxxxx`。这个接口是根据业务需要来的。

在我们应用的后端服务中，主要实现下面两个业务逻辑：
* 为前端提供一个回调地址：`GET OurHost/oauth/callback`。这个地址就是我们传给redirect_uri的。我们从query中拿到认证服务器回传给我们的code和state。然后，调用获取token的接口，获取到`access_token`。最后回调到我们的登陆接口（如下）。

* 使用token登陆的接口：`GET OurHost/login_by_token`。这个算作是具体业务场景的代码。在我们的应用中，我们通过`access_token`去请求资源服务器（认证服务器）的用户信息，然后根据这些信息自动为用户创建账号，并转跳到我们的应用页面中去。

### 实际
说了这么多，实际对接可不是一帆风顺。深刻感受到没有标准的AUTH，只有流水的实现。。。
这里，在认证服务器转跳回我们的回调地址时，首次登陆并不是服务端直接返回302，它做了一个前端转跳，导致我们的回调地址还要区分这个请求是不是异步请求；同时，还引入了跨域问题。

在解决跨域问题时，发现预检请求不会被controller捕获到。原来，我只注册了GET方法，并没有注册OPTIONS方法。虽然以前也记录过预检请求也是一种请求，但是第一次操作还是忘记了注册。

``` js
const app = new Koa();
// 注册中间件处理OPTIONS
app.use(async (ctx, next) => {
  if (ctx.method === 'OPTIONS') {
    ctx.set('Access-Control-Allow-Origin', ctx.get('Origin'));
    ctx.set('Access-Control-Allow-Headers', 'Content-Type, Content-Length, Authorization, Accept, X-Requested-With, X-XSRF-TOKEN');
    ctx.set('Access-Control-Allow-Methods', ctx.method);
    ctx.set('Access-Control-Allow-Credentials', true);
    return ctx.body = 200;
  }
  await next();
});
app.use(router.routes()); // 按给的方法注册，GET方法只注册GET的路由
app.use(router.allowedMethods()); // 处理一些未注册方法，一开始进了这里

// controller中
async oauth2Callback(ctx) {
  try {
    const isAjax = ctx.get('X-Requested-With') === 'XMLHttpRequest';
    if (isAjax) {
      setCORSHeader(ctx);
    }
    // ...
  }
}
```
别忘了再回调请求中，判断出是异步请求，也要同时返回CORS请求头。这可以用`X-Requested-With`请求头判断。

### 参考
[What is the OAuth 2.0 Authorization Code Grant Type?](https://developer.okta.com/blog/2018/04/10/oauth-authorization-code-grant-type)