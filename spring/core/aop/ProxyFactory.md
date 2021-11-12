# ProxyFactory

---

[TOC]

---

## 概述

ProxyFactory 是 Spring 中创建动态代理的工厂类，以下是 ProxyFactory 的类图：

<img src="/home/chen/_note/pic/image-20210307232919119.png" alt="image-20210307232919119" style="zoom:67%;" />



## ProxyConfig

该类中维护了一些基本的代理属性。

<img src="/home/chen/_note/pic/image-20210307233056966.png" alt="image-20210307233056966" style="zoom:67%;" />

这五个属性的含义分别是：

|      变量名      |                             作用                             |
| :--------------: | :----------------------------------------------------------: |
| proxyTargetClass |             是否直接代理类，为 false 会代理接口              |
|     optimize     | 代表子类是否能被转换为 Advised 接口，默认为 false，表示可以  |
|   exposeProxy    | 是否暴露代理，也就是是否把当前代理对象绑定到 AopContext 的 ThreadLocal 属性 currentProxy上去，常用于代理类里面的代理方法需要调用同类里面另外一个代理方法的场景 |
|      frozen      |           是否被冻结，如果被冻结，配置将不能被修改           |
|     optimize     |                       是否执行某些优化                       |



## 代理的创建流程

代理的创建流程就是从 ProxyFactory#getProxy 开始的。

<img src="/home/chen/_note/pic/image-20210307233632767.png" alt="image-20210307233632767" style="zoom:67%;" />

### 创建 AopProxy 对象

以下是 ProxyCreatorSupport#createAopProxy 的源码：

<img src="/home/chen/_note/pic/image-20210307233823035.png" alt="image-20210307233823035" style="zoom:67%;" />

getAopProxyFactory() 就是获取 AopProxyFactory 类，该类是 AopProxy 的工厂类，默认为 DefaultAopProxyFactory。

以下是 DefaultAopProxyFactory#createAopProxy 方法源码:

<img src="/home/chen/_note/pic/image-20210307234005228.png" alt="image-20210307234005228" style="zoom:67%;" />

创建代理类一共又两种方法：

1. **CGLIB - ObjenesisCglibAopProxy**
2. **Jdk Dynamic Proxy - JdkDynamicAopProxy**

从上述方法就能看出整个的选择逻辑：

>代理对象不需要优化，并且需要直接代理类（而不是接口），并且没有实现除了 SpringProxy 之外的任何接口，就直接创建 JdkDynamicAopProxy。
>
>如果以上三种有其一，就接着判断 目标代理类（TargetClass）是否是接口，或者本身就是代理类，也创建 JdkDynamicAopProxy ，不然就创建 ObjenesisCglibAopProxy。
>
>
>
>**简单理解，只有有实现用户接口就是采用 JdkDynamicAopProxy，代理目标是 接口 也使用 JdkDynamicAopProxy，例如 CurdRespository**



### 获取最终代理对象

在获取到 ProxyFactory 之后就调用 getProxy 方法获取最终的代理类。

<img src="/home/chen/_note/pic/image-20210307235518132.png" alt="image-20210307235518132" style="zoom:67%;" />

找到实现接口，最终还是使用 Proxy.newProxyInstance 来创建的代理类，以当前类 也就是 JdkDynamicAopProxy 作为最终的 InvocationHandler。