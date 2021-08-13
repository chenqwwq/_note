# Spring 中内置的 BeanPostProcessor

> 整理 Spring 汇总内置的 BeanPostProcessor，以及其作用，不做原理的深入解析。

| 类名称                                 | 实现方法 | 具体作用                                                 |
| -------------------------------------- | -------- | -------------------------------------------------------- |
| InitDestroyAnnotationBeanPostProcessor |          | 解析 InitializingBean 以及 DisposableBean。              |
| CommonAnnotationBeanPostProcessor      |          | 解析 @PostConstruct 以及 @PreDestroy，还有 @Resource     |
| AutowiredAnnotationBeanPostProcessor   |          | 解析 @Autowired 和 @Value，附带的还有 @Inject 和 @Lookup |
| AsyncAnnotationBeanPostProcessor       |          | 解析 @Async 注解，内部创建 AsyncAnnotationAdvisor 类     |
| AnnotationAwareAspectJAutoProxyCreator |          | 解析 @AspectJ 等 AOP 相关注解                            |

