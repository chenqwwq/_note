# JWT

---





## 概述

JWT 即 JSON Web Token，是一种由服务端签发，客户端保存的权限认证模式。



## JWT 的结构

JWT 中服务端签发的 Token，由以下三部分组成：

```txr
header.payload.signature
```

header 就是整个结构的头部，示例结构如下：

```json
{
    "typ":"JWT",
    "alg":"HS256"
}
```

typ 表示当前 Token 类型，alg 表示摘要算法类型。

payload 是用户的有效数据，可以保存一些非敏感的信息（不推荐保存敏感信息，一定要保存记得加密

```json
{
    "userId":1231,
    "userName":"chenqwwq",
    "expire":"1231311111"
}
```

类似上面的例子，payload 中也可以保存当前 Token 的相关信息，例如过期时间。

signature 表示的就是签名，是基于 header 中指定的 alg 算法做的摘要信息，可以按照如下形式计算：

```java
alg(base64Encode(header)+"."+base64Encode(payload))
```

服务端可以根据摘要信息判断 Token 是否被篡改。



## JWT 的缺点

JWT 的失效比较难扩展，服务端很难主动将 Token 失效掉，除非每次都向用户中心（认证中心）校验。

另外其安全性很难保证，基本的数据都是简单的 BASE64 加密（所以不能用于保存隐私信息