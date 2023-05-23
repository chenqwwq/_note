# SSO 





## 概述

**单点登录（Single Sign On）**，是指在同个授信域下共用同一套认证逻辑，只需要登录一次就可以访问全部的域内应用。





## TODO

- [ ] 各类单点登录的实现
- [ ] Tomcat 中的 Session 认证
- [ ] JWT / JWS/ JWE
- [ ] 独立的 SSO 登录系统（CAS





Tomcat 实现的 Web 应用，会在服务端，在应用层级保存用户的 Session，通过浏览器的 Cookie 来传递 SessionId（或者类似

Cookie 在浏览器中被限制只有同个域名下可以获取，比如 a.b.com 和 c.b.com 之间不能互相获取 Cookie，但是都可以获取 b.com 下到 Cookie。

