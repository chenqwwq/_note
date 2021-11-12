# Feign

> 基于 spring-cloud-openfeign-core-2.2	.4.RELEASE



---

[TOC]

---

## 整体架构图

![](assets/feign 整体架构.png)

[图片来源博客 - Feign原理 （图解）](https://www.cnblogs.com/crazymakercircle/p/11965726.html)

## 一、FeignClient 注解

> 该注解就是用来声明一个远程服务的，基于该注解代理整个接口为一个服务调用类。

<img src="assets/feignclients注解属性列表.png" alt="image-20210317212959080" style="zoom:67%;" />

| 属性名          | 属性含义                                                     |
| --------------- | ------------------------------------------------------------ |
| value           | 服务名称，如果采用 ribbon 会使用该服务名称从注册中心拉取地址 |
| name            | 等同于 value                                                 |
| contextId       | 上下文 Id，默认为服务名称                                    |
| qualifier       | 服务别名                                                     |
| url             | 请求的绝对URL，例如微信的接口，url 就可以定义为：https://api.weixin.qq.com/ |
| decode404       | 对于404异常是否需要解码                                      |
| fallback        | 降级策略，继承定义的接口方法实现就是服务降级的逻辑，实现类需要注册为Bean |
| fallbackFactory | 降级策略工厂，继承定义的接口方法实现就是服务降级的逻辑       |
| path            | 请求地址的统一前缀                                           |
| primary         | 等同于 @Primary 是不是主要的 Bean 对象，默认为 true          |

<br>



## 相关组件

### FeignContext 

FeignContext 是整个 Feign 的核心，保存着每个服务对应的 ApplicationContext，并且该 ApplicationContext 以 SpringBoot 的 ApplicationContext 为父上下文。

> 为了每个服务调用之间的配置隔离，SpringCloud 会为每个 FeignClient 生成一个 ApplicationContext，包括 Client 在内的组件都会尝试从这个上下文中取（**如果获取不到则会继续向上获取**·）。



### Feign

Feign 的创建接口，核心方法其实只有一个。

![Feign#newInstance](assets/image-20210831224552675.png)

**newInstance 方法用于根据 Contract 创建一个 HTTP API 的代理对象。**

Feign 中的默认实现是 ReflectiveFeign，并且 Feign 的抽象类定义中就包含了默认的 Builder（建造者模式）。

![Feign$Builder#build](assets/image-20210831224759975.png)

Builder 中持有了大部分的常规配置，包括 Contract 以及 Encode / Decoder，并且基本都是默认配置。

> 源码中好像很少直接调用 Fiegn$Builder#build 方法，而是调用另外一个 target 方法。

![image-20210831231920042](assets/image-20210831231920042.png)

通过 build 创建出 Feign 之后，直接 newInstance 出代理对象。

### Targeter

![image-20210831231242522](assets/image-20210831231242522.png)

> （其实我也不是很理解 Targeter 的作用。

明面上就是接收 FeignClientFactoryBean 、Feign.Builder 、FeignContext、Tareget 等参数，创建一个最终可用的代理对象，所以简单的理解是一个串联的类。

**Targeter 整合现有资源创建了最终的代理对象。**

targeter 的默认实现如下：

![image-20210831231612063](assets/image-20210831231612063.png)

就是调用 Feign.Builder#target 方法。（甚至好几个参数没用上

另外是在 Hystrix 整合的时候，Targeter 通过判断 Feign$Builder 的类型来抽取出  Fallback 相关配置，并创建各自对象。



### Target

**Target 是对被代理对象的封装。**

![image-20210831232157202](assets/image-20210831232157202.png)

三个方法返回**被代理的接口，服务名称，请求地址**。

**Target 的默认实现是 HardCodedTarget，使用 RequestTemplate 返回 Request 对象。**



### Encoder / Decoder

Feign 的编解码。



### Contract

FeignClient 接口中各方法的上的注解解析。



### Client 

Cleint 接口用于执行请求，并返回结果。

![Feign-Client接口](assets/image-20210810002230979.png)

每个 FeignClient 实例都会有一个 Client 的实现，然后由上层将请求包装为 Request 和 Options 的形式进行调用。

**Feign 使用的是装饰器模式**，层层包装完成不同的功能，例如 Ribbon 使用 LoadBalanceFeignClient 完成负载均衡的功能。





### InvocationHandler

> InvocationHandler 严格来说并不算是 SpringCloud 中的相关组件。

InvocationHandler 就是 JDK 中，对于动态代理的实现的关键，Feign 的基本原理就是通过对接口的动态代理转发接口的调用请求，所以 InvocationHandler 也是非常关键的一环。

**Feign 中默认的是  ReflectiveFeign.FeignInvocationHandler，**该类通过工厂模式创建，也就是 InvocationHandlerFactory。





### MethodHandler

![image-20210831222209078](assets/image-20210831222209078.png)

Feign 基于动态代理完成的请求转发也是间接的，**InvocationHandler 完成的只是 Method 到 MethodHandler 的映射和转发调用。**

内部还会对每个 FeignClient 中的每个方法都会进行包装，将每个方法包装为一个 MethodHandler。

**MethodHandler 的默认实现是 SynchronousMethodHandler。**

类和 MethodHandler 在 InvocationHandler 中以 dispatch（Map<Method,MethodHandler>）的形式保存。



> 请求调用的粗略过程：

使用 Feign 的调用，首先通过的是接口的 InvocationHandler，通过 Map 映射获取到对应的 MethodHandle



<br>



> 老子发现 Feign 很喜欢使用建造者模式或者工厂模式，而且写在一个类里面。
>
> ReflectiveFeign 的创建方法竟然在 Feign$Builder 里面，

## 二、Bean 注册流程

> 注册流程就是解析 FeignClient 并且创建 Bean 对象的过程。
>
> 简单可以分为如下步骤:
>
> 1. 定位 FeignClient 接口
> 2. 解析 FeignClient 注解
> 3. **注册 BeanDefinition**

<br>

## 三、代理创建流程

> 在 Bean 的注册流程可以发现，最终注册的 Bean 对象为 FeignClientFactoryBean 类型。
>
> FactoryBean 接口就是 Spring 对**工厂方法模式的实现**，所有的创建流程包含在 getObject() 方法中。
>
> **特别需要关注的就是 InvocationHandler 的具体实现（FeignClient 是通过 JDK 动态代理实现的代理模式）。**

<br>

## 四、默认服务调用流程

> 通过 JDK 动态代理创建了真实的代理对象之后，具体的服务调用过程。
>
> 基本的 Http 调用以及 **LoadBalance 调用（和 Ribbon 的整合）**，以及**和 Hystrix 的整合过程**。
>
> 有一个需要关注的点就是，Feign 如何映射真实接口方法和具体的功能实现。

<br>

## 相关参考

[Feign 终级解析](https://mp.weixin.qq.com/s?__biz=MzUwOTk1MTE5NQ==&mid=2247483724&idx=1&sn=03b5193f49920c1d286b56daff8b1a09&chksm=f90b2cf8ce7ca5ee6b56fb5e0ffa3176126ca3a68ba60fd8b9a3afd2fd1a2f8a201a2b765803&token=302932053&lang=zh_CN&scene=21#wechat_redirect)

[Feign原理 （图解）](https://www.cnblogs.com/crazymakercircle/p/11965726.html)

[OpenFeign与Ribbon源码分析总结与面试题](https://juejin.cn/post/6844904200841740302)

[Spring Cloud OpenFeign源码分析](https://blog.csdn.net/baidu_28523317/article/details/106935370)