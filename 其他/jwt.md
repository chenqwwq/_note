# JWT

---

[TOC]

---

## 概述

```
// JSON 官网的标语
JSON Web Tokens are an open, industry standard RFC 7519 method for representing claims securely between two parties.
```



JWT 即 JSON Web Token，**是一种基于 Json 格式的信息交换方式。**

<br>

JWT 可以分为 JWS / JWE 两种模式，国内使用的大部分都是 JWS，相对于 JWS 来说 JWE 内容更多，加密流程也更复杂，所以也更加安全。

（但是 JWS 也是可以信任的，因为包含了摘要算法可以防止篡改，但敏感数据不建议放在 JWS 中。

（好像网上的博客都直接把 JWS 当作 JWT 了，emmm



JWT 的使用场景包含权限认证（Authentication）以及信息交换（Exchanger）。



## JWS 的结构

JWT 中服务端签发的 Token，由以下三部分组成：

```txt
header.payload.signature   // 三部分使用 . 分割
```

header 就是整个结构的头部，示例结构如下：

```json
{
    "typ":"JWT",
    "alg":"HS256"
}
```

typ 表示当前 Token 类型，alg 表示摘要算法类型，例如 HMAC(HMAC-SHA256) ，RS256(RSA-SHA256)  等等。

<br>

payload 是用户的有效数据，可以保存一些非敏感的信息（不推荐保存敏感信息，一定要保存记得加密

```json
{
    "userId":1231,
    "userName":"chenqwwq",
    "expire":"1231311111"
}
```

类似上面的例子，payload 中也可以保存当前 Token 的相关信息，例如过期时间。

对于 payload 在 RFC 文档中规定了部分的字段，例如：

- iss - Issuer，签发机构

- exp - expiration time，过期时间
- sub - subject，主体信息
- aud - aduience

<br>

signature 表示的就是签名，是基于 header 中指定的 alg 算法做的摘要信息，可以按照如下形式计算：

```java
alg(base64Encode(header)+"."+base64Encode(payload),secret_key) 
```

alg 表示的就是在 header 中定义的摘要算法，这里我们定义的是 HS256，另外还有部分场景可能不需要添加密钥。



服务端可以根据摘要信息判断 Token 是否被篡改。



## JWT 的缺点

JWT 的失效比较难扩展，服务端很难主动将 Token 失效掉，除非每次都向用户中心（认证中心）校验。

另外其安全性很难保证，基本的数据都是简单的 BASE64 加密（所以不能用于保存隐私信息





## 引用

- [JWT - RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1)
- [JWS - RFC 7515](https://www.rfc-editor.org/rfc/rfc7515.html)