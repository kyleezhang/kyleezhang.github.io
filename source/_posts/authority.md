---
title: 关于身份认证与鉴权方案整理及思考
date: 2021-09-18 10:59:06
toc: true
mathjax: false
categories: 
- 服务端
tags:
- 登录鉴权
---

## 认证方式

### 一、基于会话 Cookie+Session

> 用户使用账户密码成功登录某个服务后，服务端创建一个 session 对象，并把 sessionid set cookie 给客户端，最常用的就是KV存储。由于 cookie 在每一次请求时会默认带上，一般来说前端感知较低

首先我们来看 session，在计算机科学中，会话（Session）指多个通信实体之间的一次临时和互动的信息交换。会话通常是有状态的，这意味着通信的实体中至少有一方需要保存当时的状态信息。网络应用程序（Web Application）的客户端和服务端一般基于 HTTP 进行通信，之间的交互可以看成会话，即 HTTP 会话。众所周知，HTTP 是无状态的，即没有在协议层实现状态信息的存储和同步，那 HTTP 会话如何实现状态信息的存储和同步？

<!-- more -->

HTTP 状态管理机制（[State Management Mechanism](https://tools.ietf.org/html/rfc6265)）定义了 HTTP 头字段 Cookie 和 Set-Cookie，以及如果通过这两个头字段去实现状态管理。但是由于各种历史原因，Cookie 采用明文存储和传输，且不对数据的完整性进行校验，保密性和完整性都比较弱。Cookie 一般用来存储状态信息，而状态信息一般属于敏感数据。这样一来，敏感数据就被暴露在一个不安全的环境中，因此更安全的选择是将状态信息交由服务端保存，这也就产生了 Cookie + Session 的解决方案。

关于 Cookie 我们还需要注意以下几点：

#### 1、Cookie 的生命周期

Cookie 的生命周期可以通过两种方式定义：

- 会话期 Cookie 是最简单的 Cookie：浏览器关闭之后它会被自动删除，也就是说它仅在会话期内有效。会话期Cookie不需要指定过期时间（Expires）或者有效期（Max-Age）。需要注意的是，有些浏览器提供了会话恢复功能，这种情况下即使关闭了浏览器，会话期Cookie 也会被保留下来，就好像浏览器从来没有关闭一样，这会导致 Cookie 的生命周期无限期延长。
- 持久性 Cookie 的生命周期取决于过期时间（Expires）或有效期（Max-Age）指定的一段时间。不过需要注意的是当 Cookie 的过期时间被设定时，设定的日期和时间只与客户端相关，而不是服务端。

#### 2、限制访问 Cookie

有两种方法可以确保 Cookie 被安全发送，并且不会被意外的参与者或脚本访问：Secure 属性和HttpOnly 属性。

标记为 Secure 的 Cookie 只应通过被 HTTPS 协议加密过的请求发送给服务端，因此可以预防 [man-in-the-middle（中间人攻击）](https://developer.mozilla.org/zh-CN/docs/Glossary/MitM) 攻击者的攻击。但即便设置了 Secure 标记，敏感信息也不应该通过 Cookie 传输，因为 Cookie 有其固有的不安全性，Secure 标记也无法提供确实的安全保障, 例如，可以访问客户端硬盘的人可以读取它。

JavaScript Document.cookie API 无法访问带有 HttpOnly 属性的cookie；此类 Cookie 仅作用于服务器。例如，持久化服务器端会话的 Cookie 不需要对 JavaScript 可用，而应具有 HttpOnly 属性。此预防措施有助于缓解跨站点脚本（XSS） (en-US)攻击。

#### 3、Cookie 的作用域

Domain 和 Path 标识定义了Cookie的作用域：即允许 Cookie 应该发送给哪些URL。

**Domain 属性**

Domain 指定了哪些主机可以接受 Cookie。如果不指定，默认为 origin，不包含子域名。如果指定了Domain，则一般包含子域名。因此，指定 Domain 比省略它的限制要少。但是，当子域需要共享有关用户的信息时，这可能会有所帮助。 

例如，如果设置 Domain=mozilla.org，则 Cookie 也包含在子域名中（如developer.mozilla.org）。

**Path 属性**

Path 标识指定了主机下的哪些路径可以接受 Cookie（该 URL 路径必须存在于请求 URL 中）。以字符 %x2F ("/") 作为路径分隔符，子路径也会被匹配。

例如，设置 Path=/docs，则以下地址都会匹配：

/docs
/docs/Web/
/docs/Web/HTTP

#### 4、SameSite

SameSite Cookie 允许服务器要求某个 cookie 在跨站请求时不会被发送，从而可以阻止跨站请求伪造攻击（CSRF）。

SameSite 可以有下面三种值：

- None。浏览器会在同站请求、跨站请求下继续发送 cookies，不区分大小写。
- Strict。浏览器将只在访问相同站点时发送 cookie。（在原有 Cookies 的限制条件上的加强，如上文 “Cookie 的作用域” 所述）
- Lax。与 Strict 类似，但用户从外部站点导航至URL时（例如通过链接）除外。 在新版本浏览器中，为默认选项，Same-site cookies 将会为一些跨站子请求保留，如图片加载或者 frames 的调用，但只有当用户从外部站点导航到URL时才会发送。如 link 链接

> 以前，如果 SameSite 属性没有设置，或者没有得到运行浏览器的支持，那么它的行为等同于 None，Cookies 会被包含在任何请求中——包括跨站请求。但目前大多数主流浏览器正在将 SameSite 的默认值迁移至 Lax。如果想要指定 Cookies 在同站、跨站请求都被发送，现在需要明确指定 SameSite 为 None。

### 二、基于 Token

> 用户登录成功后，服务端会用密钥签发 Token，客户端每次向服务的发送请求都会携带 Token，当服务的接收到 Token 时会用密钥去验证是否有效，从而实现身份验证

Token格式不限，JSON Web Token（JWT）是目前较为通用的 Token 认证方式，因此我们以 JWT 为例展开介绍。

上面讲过，Cookie 的弱保密性和弱完整性会引发安全问题。如果对 Cookie 中的数据进行加密和完整性校验，安全性会不会大幅度提升？JWT（JSON Web Token）就是这样一种解决方案，可以用来解决 Cookie 的安全问题。

实际的 JWT 大概就像下面这样：

<img src="/assets/authority/01.png">

它是一个很长的字符串，中间用点（.）分隔成三个部分。注意，JWT 内部是没有换行的，这里只是为了便于展示，将它写成了几行。三个部分分别为：

- Header
- Payload
- Signature

#### 1、Header

Header 部分是一个 JSON 对象，描述 JWT 的元数据，通常是下面的样子。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

上面代码中，alg属性表示签名的算法（algorithm），默认是 HMAC SHA256（写成 HS256）；typ属性表示这个令牌（token）的类型（type），JWT 令牌统一写为JWT。最后，将上面的 JSON 对象使用 Base64URL 算法转成字符串。

#### 2、Payload

Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段，供选用。

- iss (issuer)：签发人
- exp (expiration time)：过期时间
- sub (subject)：主题
- aud (audience)：受众
- nbf (Not Before)：生效时间
- iat (Issued At)：签发时间
- jti (JWT ID)：编号

除了官方字段，你还可以在这个部分定义私有字段，例如：

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

注意，JWT 默认是不加密的，任何人都可以读到，所以不要把秘密信息放在这个部分。这个 JSON 对象也要使用 Base64URL 算法转成字符串。

#### 3、Signature

Signature 部分是对前两部分的签名，防止数据篡改。

首先，需要指定一个密钥（secret）。这个密钥只有服务器才知道，不能泄露给用户。然后，使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。

```js
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

算出签名以后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用"点"（.）分隔，就可以返回给用户。

#### 4、Base64URL

前面提到，Header 和 Payload 串型化的算法是 Base64URL。这个算法跟 Base64 算法基本类似，但有一些小的不同。

JWT 作为一个令牌（token），有些场合可能会放到 URL（比如 api.example.com/?token=xxx）。Base64 有三个字符+、/和=，在 URL 里面有特殊含义，所以要被替换掉：=被省略、+替换成-，/替换成_ 。这就是 Base64URL 算法。
