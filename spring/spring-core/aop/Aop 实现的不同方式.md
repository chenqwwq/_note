# AOP 实现的不同方式



## Feign

Feign 中也是通过代理类来完成接口调用功能。

Feign 的代理类通过 FeignClientsRegistrar 来完成注册的流程，FeignClientRegistrar 继承于 ImportBeanDefinitionRegistrar。

> ImportBeanDefinitionRegistrar 类会在 ConfigurationClassPostProcessor 调用时被解析，是在创建所有的 BeanPostProcessor 之前。
>
> ConfigurationClassPostProcessor 继承于 BeanDefinitionRegistryPostProcessor，会在 BeanFactory 初期就被调用。
>
> BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry() 就是用来修改当前应用上下文的注册列表，可以增加删除或者修改相关的 BeanDefinition，此时 Bean 并未创建。

FeignClientRegistrar 会扫描指定的包（默认为当前根目录），获得所有的 FeignClient 标注的接口，然后解析并注册为 Bean 对象。

> 所有的 FeignClient 最终都是被注册为 FeignClientFactoryBean。
>
> FeignClientFactoryBean#getObject 中







## Async

同样是代理，Async 使用的是 BeanPostProcessor 来完成代理类的创建。

AsyncAnnotationBeanPostProcessor 继承了AbstractAdvisingBeanPostProcessor，该抽象类就是带着 Advisor 的 BeanPostProcessor。

> 也就是说 Async 并不是使用 Spring 官方提供了创建流程，它的 Advisor 并不会注册到 Spring 容器。

AbstractAdvisingBeanPostProcessor 对实例化后，初始化的 Bean 进行拦截，使用 ProxyFactory 创建对应的代理类。

> 不采用和 Feign 一样直接扫描全部类的注册流程，主要应该还是因为 Async 注解定义在方法上，无法快速扫描。









## AbstractAuthProxyCreator

该类继承继承了
