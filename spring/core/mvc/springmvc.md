# Spring MVC



---

[TOC]



## Spring MVC 组件

### DispatcherServlet 

Spring MVC 是基于 Servelt 实现的服务端框架，DispatcherServlet 就是前端的处理器，由它负责调度到真实的业务方法。

<br>

另外 DispatcherServlet 也是各个组件的初始化和串联工具，初始化流程如下：

![image-20211110202955634](assets/image-20211110202955634.png)

（Spring MVC 包含哪些组件从上面就能看出来了，HHH。

<br> 

### HandlerMapping 

HandlerMapping 就是 Handler（处理器）的映射类，定义的接口中只有一个方法，通过 Request 参数获取 HandlerExecutionChain（处理器执行链）。

![image-20211014134242860](assets/image-20211014134242860.png)

**HandlerMapping 根据请求找到具体的 Handler，并且和所有匹配的 HandlerInterceptor 串联，组成 HandlerExecutionChain。**

不同的实现类有不同的匹配方式，常见的实现有 RequestMappingHandlerMapping 以及 SimpleUrlHandlerMapping。

![](assets/HandlerMapping.jpg)

[图片来源文章](https://www.jianshu.com/p/f04816ee2495)

HandlerMapping 在 DispatcherServlet 中初始化，从入参的 ApplicationContext 中获取所有的 HandlerMapping 类型的 Bean 对象。

<br>

![](assets/AbstractHandlerMapping.png)

AbstractHandlerMapping 是 HandlerMapping 最直接的实现，实现了基本的 HandlerExecutionChain 封装逻辑。

> AbstractHandlerMapping 中同时保存了拦截器的集合。

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 子类实现具体的获取方法
    Object handler = getHandlerInternal(request);
    // 获取默认的 Handler
    if (handler == null) {
        handler = getDefaultHandler();
    }
    // 没有就退出
    if (handler == null) {
        return null;
    }
    // Bean name or resolved handler?
    // 返回的是 Bean 名称
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }
    // 封装为 HandlerExecutionChain，该过程中会尝试匹配所有的 HandlerInterceptor
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    // 处理跨域等问题
    if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
        CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        config = (config != null ? config.combine(handlerConfig) : handlerConfig);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
}
```

AbstractUrlHandlerMapping 是根据 UrlPath 搜索 Handler 的实现，例如早期的 XML 时代，需要指定匹配的路径，类中使用 Map 保存了 Url 到 Handler 的映射关系。（Handler 一般就是 Controller 接口的实现。

常用的实现有 SimpleUrlHandlerMapping 以及 BeanNameUrlHandlerMapping。

另外就是 AbstractHandlerMethodMapping，该类使用 MappingRegistry 保存具体的映射关系，获取的就是 HandlerMethod 对象，泛型的 Key。

常用的比如 RequestMappingHandlerMapping，使用 RequestMappingInfo 作为 Key，保存了 @RequestMapping 的各类参数。



### HandlerAdapter

处理器执行器，各类的执行器会组成类似责任链模式的逻辑，遍历判断是否可以处理 Handler（supports 方法），可以的话进一步处理（handler 方法）。

![image-20211014135346042](assets/image-20211014135346042.png)

HandlerAdapter 就是针对各类的 Handler 的适配，如果你实现了自定义的 Handler，就一定需要自定义的 HandlerAdapter 去执行。

**实现的责任链模式中，使用 supports 方法判断当前类是否可以处理 Handler（入参），可以则进一步调用 handler 执行。**

## 参数解析

### 

## HandlerInterceptor

处理器的拦截器接口，包含了如下三个方法：

| 方法签名        | 调用时机                                  | 备注                                           |
| --------------- | ----------------------------------------- | ---------------------------------------------- |
| preHandle       | 前置处理器，在调用真实的 Handler 之前执行 |                                                |
| postHandle      | 在 Handler 执行完毕，并且渲染之前执行     | 入惨包含 ModelAndView 可以对视图进行修改       |
| afterCompletion | 渲染完成之后执行                          | 异常未被处理时该方法入参携带异常（处理了就没了 |

在 DispactherServlet 的中调用实现逻辑如下：

![](../../../../Desktop/HandlerInterceptor%E7%9A%84%E6%89%A7%E8%A1%8C%E6%97%B6%E6%9C%BA.png)

（后面三个都是对 AfterCompletion 的调用。

常用的基础实现有 MappedInterceptor，提供了对于路径 Include 以及 Exclude 的功能。



## Spring MVC 的处理流程

![/home/chen/Desktop/RequestLifecycle.png](/home/chen/Desktop/RequestLifecycle.png)





## 总结

Spring MVC 的实现中最明显的特色就是责任链模式，在以下场景中都使用到了该模式：

1. Handler 的执行，会在 HandlerAdapter 的链中逐个查找
2. HandlerInterceptor，拦截器的实现整个就是一个链吧
3. 参数的解析，例如 HandlerMethodArgumentResolver，对于接口入参的解析

责任链模式的优点还是在于真实对象和处理的解耦，将一个对象扔进链中就好，不必关心真实的处理逻辑。

