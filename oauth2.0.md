# OAuth 2.0 之 Authorization code 与 Implicit

OAuth 2.0 是用于授权的行业标准协议。我们常见的比如第三方登录、授权第三方应用获取保存在其它服务商的个人数据，这种都是 OAuth 2.0 的应用场景。

OAuth 2.0 有四种授权方式：
- `Authorization code`
- `Implicit`
- `Resource Owner Password Credentials`
- `Client Credentials`

本文将会讲述其中的两种授权方式 `Authorization code` 和 `Implicit`。


## 基本概念

### 角色

OAuth 定义了以下四种角色：
- `resource owner` : 资源所有者，授予第三方客户端访问权限的实体。
- `resource server` : 资源服务器，托管和保护资源的服务器，对持有访问令牌的资源请求做出响应。
- `client` : 客户端，代表资源所有者并在其授权下请求受保护的资源。
- `authorization server` : 授权服务器，认证资源所有者并在其授权下向客户端颁发访问令牌（ `access token` ）。


例如我们使用微信登录某个第三方论坛，这时：
- `resource owner` 资源所有者就是拥有微信账号的我们个人。
- `resource server` 资源服务器则是微信的服务器，因为这里请求的资源其实是微信托管和保护的用户资源。
- `client` 客户端就是这个第三方论坛。
- `authorization server` 授权服务器也是微信的服务器，具体来说就是其中专门负责处理授权的模块或者服务。

你可以发现 `resource server` 和 `authorization server` 其实是紧密相关的，如果服务器是单体架构，那么这两者就是同一个服务中的不同模块而已。

### `Access Token` & `Refresh Token`

`Access Token` ： 访问令牌，持有 `access token` 就等于得到了用户授权从而可以访问某些资源，因为安全方面的考虑，`access token` 的有效时间一般较短。

`Refresh Token` ：刷新令牌，使用 `refresh token` 可以获取新的 `access token`，`refresh token` 一般具有更长的有效期，这么做可以避免因为 `access token` 过期而导致需要用户频繁重新授权的情况。


### 客户端注册

例如我们要想使用微信登录某个第三方论坛，那么这个第三方论坛必须得先向微信那边去注册一下，表明自己是一个合法的第三方客户端，并且获取需要在后续授权过程中使用的一些参数。

在开始 OAuth 授权流程之前，客户端必须先完成注册，注册时一般需要提交以下信息：
- 声明客户端类型。
- 提供客户端重定向的 URI 。
- 其它信息，比如应用名称、网站地址、描述、logo、法律条款等等。

客户端注册完成后，一般会获取到以下两个参数：
- `client_id`
- `client_secret`

这两个参数在后续的授权过程中将会用到。


下面将会正式开始介绍 `Authorization code` 和 `Implicit` 这两种授权方式的整个过程。

## Authorization code

`Authorization code` 授权码方式，这种方式最常用且安全性也很高。

![Authorization code](https://raw.githubusercontent.com/RifeWang/images/master/oauth-authorization-code-flow.png)

如上图所示，整个过程为：

1、client 发起授权请求，例如：

```
GET /authorization?
    client_id=12345&
    redirect_uri=https://client-app.com/callback&
    response_type=code&
    scope=openid%20profile&
    state=ae13d489bd00e3c24

Host: oauth-authorization-server.com
```

请求包含以下参数：
- `client_id` ：客户端注册后获取的唯一值。
- `redirect_uri` ：重定向地址。
- `response_type` ：值为 `code` 表明授权方式是授权码。
- `scope` ：访问数据的作用域。
- `state` ：当前会话的唯一值。

2、用户登录并确认授权。

3、浏览器重定向到客户端 `redirect_uri` 地址，并返回 `code` 授权码，例如：

```
GET /callback?
    code=a1b2c3d4e5f6g7h8&
    state=ae13d489bd00e3c24

Host: client-app.com
```

其中的 state 与发起请求的 state 应该一致。

前三步都是在浏览器环境中进行的，后面的步骤则是客户端服务器与 OAuth 服务器之间直接进行通信。

4、客户端发起请求，使用 `code` 交换 `access token`，例如：

```
POST /token
Host: oauth-authorization-server.com
…
client_id=12345&
client_secret=SECRET&
redirect_uri=https://client-app.com/callback&
grant_type=authorization_code&
code=a1b2c3d4e5f6g7h8
```

请求参数：
- `client_id` ：客户端注册后获取的 client_id。
- `client_secret` ：客户端注册后获取的 client_secret。
- `redirect_uri` ：客户端重定向地址。
- `grant_type` ：声明授权类型为 `authorization_code`。
- `code` ：上一步获取的授权码。

5、OAuth 服务器生成 `access token` 并响应数据，例如：

```
{
    "access_token": "z0y9x8w7v6u5",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "openid profile",
    ...
}
```
响应数据可能还包括 `refresh token` 。

6、客户端调用 API 请求资源，例如：
```
GET /userinfo HTTP/1.1
Host: oauth-resource-server.com
Authorization: Bearer z0y9x8w7v6u5
```

7、资源服务器响应：
```
{
    "username": "username",
    "email": "123@email.com",
    ...
}
```

以上就是一个完整的授权码授权并获取数据的过程。


## Implicit

`Implicit` 称之为简化模式或者隐藏模式，过程更加简单，但安全性也有所降低。

![Implicit](https://raw.githubusercontent.com/RifeWang/images/master/oauth-implicit-flow.png)

如上图所示，整个过程为：

1、 client 发起授权请求，例如：
```
GET /authorization?
    client_id=12345&
    redirect_uri=https://client-app.com/callback&
    response_type=token&
    scope=openid%20profile&
    state=ae13d489bd00e3c24

Host: oauth-authorization-server.com
```

请求包含以下参数：
- `client_id` ：客户端注册后获取的唯一值。
- `redirect_uri` ：重定向地址。
- `response_type` ：值为 `token` 表明授权方式是简化模式。
- `scope` ：访问数据的作用域。
- `state` ：当前会话的唯一值。

2、用户登录并确认授权。

3、直接生成 `access token` 并重定向到客户端地址：
```
GET /callback#
    access_token=z0y9x8w7v6u5&
    token_type=Bearer&
    expires_in=5000&
    scope=openid%20profile&
    state=ae13d489bd00e3c24

Host: client-app.com
```

重定向使用的是 `#` 连接参数，而不是 query 参数，这是因为浏览器对 URI 发起请求时不会携带 `#` 符号之后的数据，使用 `#` 也是安全方面的考虑。

4、客户端调用 API 请求资源。

5、资源服务器响应。


以上就是一个简化模式的整个过程。


## 结语

本文介绍了 OAuth 2.0 中的 `Authorization code` 和 `Implicit` 两张授权方式，前者最常用也更安全，后者更方便但安全性更低。


![公众号](https://raw.githubusercontent.com/RifeWang/images/master/qrcode.jpg)
