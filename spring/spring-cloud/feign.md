# Feign



---



## Feign 对象的注册过程

Feign 的启动注解为 EnableFeignClients，标记该注解之后借由 Import 注解，开始执行 FeignClientsRegistrar 类。·

<img src="/home/chen/github/_note/pic/image-20210301224037710.png" alt="image-20210301224037710" style="zoom:67%;" />

> Import 注解由 ConfigurationClassPostProcessor 类解析。







## 文章引用

[Feign终级解析](https://mp.weixin.qq.com/s?__biz=MzUwOTk1MTE5NQ==&mid=2247483724&idx=1&sn=03b5193f49920c1d286b56daff8b1a09&chksm=f90b2cf8ce7ca5ee6b56fb5e0ffa3176126ca3a68ba60fd8b9a3afd2fd1a2f8a201a2b765803&token=302932053&lang=zh_CN&scene=21#wechat_redirect)

