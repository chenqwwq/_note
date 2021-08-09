# Resource 和 Autowired



## Resource 和 Autowired 的区别

Autowired 是 Spring 官方提供的注入方法，**默认使用 byType 注入**，可以和 @Qualifier 结合，指定 Bean 的名称，可以通过 **required=false 指定非必要**。

> Autowired 使用 AutowiredAnnotationBeanPostProcessor 实现。 
>
> 该类同时实现的还有以下注解：
>
> 1. @Value
> 2. @Autowired
> 3. @Inject（非 Spring 官方提供）

Resource 并不是官方提供的注解，同时提供了 name 和 type 两种方式，如果都不指定默认为 name，也可以同时指定。

> Resource 使用 CommonAnnotationBeanPostProcessor 实现。
>
> 该类同时实现的还有以下注解：
>
> 1. @Resource（非 Spring 官方提供）
> 2. @PostConstruct（非 Spring 官方提供）
> 3. @PreDestory（非 Spring 官方提供）



