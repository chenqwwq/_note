# Feign

> 基于 spring-cloud-openfeign-core-2.2.4.RELEASE

---

[TOC]

---



## FeignClient 注解

> 该注解就是用来声明一个远程服务的，基于该注解代理整个接口为一个服务调用类。

<img src="/home/chen/_note/pic/image-20210317212959080.png" alt="image-20210317212959080" style="zoom:67%;" />

| 属性名          | 属性含义                                                     |
| --------------- | ------------------------------------------------------------ |
| value           | 服务名称                                                     |
| name            | 等同于value                                                  |
| contextId       | 上下文Id，默认为服务名称                                     |
| qualifier       | 服务别名                                                     |
| url             | 请求的绝对URL，例如微信的接口，url 就可以定义为：https://api.weixin.qq.com/ |
| decode404       | 对于404异常是否需要解码                                      |
| fallback        | 降级策略，继承定义的接口方法实现就是服务降级的逻辑，实现类需要注册为Bean |
| fallbackFactory | 降级策略工厂，继承定义的接口方法实现就是服务降级的逻辑       |
| path            | 请求地址的统一前缀                                           |
| primary         | 等同于 @Primary 是不是主要的Bean对象，默认为true             |





## FeignClient -  注解扫描流程

FeignClient 注解的解析流程在于 **FeignClientsRegistrar**，该类在 EnableFeignClients 注解中被加入到容器中。

该注解源码如下：

<img src="/home/chen/_note/pic/image-20210304213239994.png" alt="image-20210304213239994" style="zoom:67%;" />

@Import 中的 FeignClientsRegistrar 继承了 ImportBeanDefinitionRegistrar。

> ImportBeanDefinitionRegistrar 类借由 ConfigurationClassPostProcessor，在上下文刷新阶段就会调用该接口的  registerBeanDefinitions 方法。

<img src="/home/chen/_note/pic/image-20210304213302218.png" alt="image-20210304213302218" style="zoom:67%;" />

首先会向容器注册一个默认的配置类。

<img src="/home/chen/_note/pic/image-20210318214330736.png" alt="image-20210318214330736" style="zoom:67%;" />

从 EnableFeignClients 注解中提取 defaultConfiguration 属性，默认为空。

<img src="/home/chen/_note/pic/image-20210318214438618.png" alt="image-20210318214438618" style="zoom:67%;" />

此时如果是默认，则以空对象注册到 BeanFactory 中，配置类的类型是  FeignClientSpecification。

> 配置项中包括了 Contract，Encoder，Decoder，ConversionService 等基础的配置项。
>
> Feign 可以为每个 FeignClient 指定一个单独的配置类，如果没有配置则采用默认的配置类。



之后的 registerFeignClients 方法的作用则是扫描 FeignClient 注解，并向容器注册。

```java
// FeignClientsRegistrar#registerFeignClients
public void registerFeignClients(AnnotationMetadata metadata,
                                 BeanDefinitionRegistry registry) {
    // 获取类扫描器，它的功能就是扫描整个包的所有Java类
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    // 资源加载器
    scanner.setResourceLoader(this.resourceLoader);
    Set<String> basePackages;
    // 获取 EnableFeignClients 注解的属性
    Map<String, Object> attrs = metadata
        .getAnnotationAttributes(EnableFeignClients.class.getName());
    // 这里是类的扫描条件，被 FeignClient 标识。
    AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
        FeignClient.class);
    final Class<?>[] clients = attrs == null ? null
        : (Class<?>[]) attrs.get("clients");
    
    // 是否指定了某些需要扫描的包，没有的话直接扫描整个ClassPath
    if (clients == null || clients.length == 0) {
        scanner.addIncludeFilter(annotationTypeFilter);
        basePackages = getBasePackages(metadata);
    }else {
        final Set<String> clientClasses = new HashSet<>();
        basePackages = new HashSet<>();
        for (Class<?> clazz : clients) {
            basePackages.add(ClassUtils.getPackageName(clazz));
            clientClasses.add(clazz.getCanonicalName());
        }
        AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
            @Override
            protected boolean match(ClassMetadata metadata) {
                String cleaned = metadata.getClassName().replaceAll("\\$", ".");
                return clientClasses.contains(cleaned);
            }
        };
        scanner.addIncludeFilter(
            new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
    }

    // 针对每个包的扫描
    for (String basePackage : basePackages) {
        // 扫描得到具体的 BeanDefinition
        Set<BeanDefinition> candidateComponents = scanner
            .findCandidateComponents(basePackage);
        // 遍历被FeignClient标注的类
        for (BeanDefinition candidateComponent : candidateComponents) {
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                // verify annotated class is an interface
                AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
                // @FeignClient 只能标注在接口上
                Assert.isTrue(annotationMetadata.isInterface(),
                              "@FeignClient can only be specified on an interface");
				// 获取@FeignClient的注解属性
                Map<String, Object> attributes = annotationMetadata
                    .getAnnotationAttributes(
                    FeignClient.class.getCanonicalName());
				// 获取Feign对象的名称
                String name = getClientName(attributes);
                // 注册这个Feign对象的配置类
                registerClientConfiguration(registry, name,
                                            attributes.get("configuration"));
				// 注册FeignClient
                registerFeignClient(registry, annotationMetadata, attributes);
            }
        }
    }
}

```

整个流程就是确定扫描的目录，然后用 ClassPathScanningCandidateComponentProvider 扫描。

扫描的目录如下，如果没有指明 clients 会扫描整个根目录，如果指明了 clients，会扫描 clients 所在的包，以及 clients 表示的类。

扫描之后会为每个 FeignClient 类注册一个配置类，再注册一个 FeignClient 的代理类。

> FeignClient 只能标注在接口上。

注册配置类的逻辑上面的一样，但是此时取的是 FeignClient 中指明的配置类。

以下是注册 FeignClient 的流程:

```java
// FeignClientsRegistrar#registerFeignClient
private void registerFeignClient(BeanDefinitionRegistry registry,
                                 AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();
    // 这里定义了真实的Bean类型，就是使用 FeignClientFactoryBean
    BeanDefinitionBuilder definition = BeanDefinitionBuilder
        .genericBeanDefinition(FeignClientFactoryBean.class);
    
    // 通过注解的属性添加BeanDefinition的配置项
    validate(attributes);
    definition.addPropertyValue("url", getUrl(attributes));
    definition.addPropertyValue("path", getPath(attributes));
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    String contextId = getContextId(attributes);
    definition.addPropertyValue("contextId", contextId);
    definition.addPropertyValue("type", className);
    definition.addPropertyValue("decode404", attributes.get("decode404"));
    definition.addPropertyValue("fallback", attributes.get("fallback"));
    definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
	// 还tm有个别名
    String alias = contextId + "FeignClient";
    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
    beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);

    // has a default, won't be null
    // 默认就是Primary
    boolean primary = (Boolean) attributes.get("primary");
    beanDefinition.setPrimary(primary);
    String qualifier = getQualifier(attributes);
    if (StringUtils.hasText(qualifier)) {
        alias = qualifier;
    }
	// 组装为BeanDefinitionHolder
    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
                                                           new String[] { alias });
    // 使用工具类注册，这里就是真实名称和别名一起注册的
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

> 首先重要的一点就是，扫描的 FeignClient 类都会被注册为 FeignClientFactoryBean。

会将 FeignClient 中的所有属性都取出来塞进 BeanDefinition里面，然后为该类以及所有别名注册 Bean 对象。



> 以上就是扫描的全过程。



### 小结

Feign 借由 ImportBeanDefinitionRegistrar 接口，在容器初始化阶段扫描 FeignClint 标注的类并以 FeignClientFactoryBean 注册了 BeanDefinition。

除了默认的配置类之外，还为每一个 FeignClient 都注册了自己的配置对象。







> FeignClientFactoryBean 继承于 FactoryBean ，所以真实的代理创建流程还是在 FeignClientFactoryBean#getObject() 方法中。

## FeignClientFactoryBean -  代理对象创建流程

以下是 FeignClientFactoryBean#getObject 方法源码：

<img src="/home/chen/_note/pic/image-20210304232239220.png" alt="image-20210304232239220" style="zoom:67%;" />

内部封装了一层，直接调用的 getTarget() 方法。

```java
// FeignClientFactoryBean#getTarget
<T> T getTarget() {
    // 从 ApplicationContext 中获取 FeignContext 类Bean对象
    FeignContext context = applicationContext.getBean(FeignContext.class);
    // 获取 FeignBuilder
    Feign.Builder builder = feign(context);
	
    // 根据是否有url
    if (!StringUtils.hasText(url)) {
        if (!name.startsWith("http")) {
            url = "http://" + name;
        }
        else {
            url = name;
        }
        url += cleanPath();
        return (T) loadBalance(builder, context,
                               new HardCodedTarget<>(type, name, url));
    }
    if (StringUtils.hasText(url) && !url.startsWith("http")) {
        url = "http://" + url;
    }
    String url = this.url + cleanPath();
    Client client = getOptional(context, Client.class);
    if (client != null) {
        if (client instanceof LoadBalancerFeignClient) {
            // not load balancing because we have a url,
            // but ribbon is on the classpath, so unwrap
            client = ((LoadBalancerFeignClient) client).getDelegate();
        }
        if (client instanceof FeignBlockingLoadBalancerClient) {
            // not load balancing because we have a url,
            // but Spring Cloud LoadBalancer is on the classpath, so unwrap
            client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
        }
        builder.client(client);
    }
    Targeter targeter = get(context, Targeter.class);
    return (T) targeter.target(this, builder, context,
                               new HardCodedTarget<>(type, name, url));
}

```

以下是 feign 也就是拼装 Feign.Builder 的过程:

```java
// FeignClientFactoryBean#feign
protected Feign.Builder feign(FeignContext context) {
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    Logger logger = loggerFactory.create(type);

    // @formatter:off
    // 根据配置类，如果开启Hystrix的话，此处就为HystrixFeign.builder()
    Feign.Builder builder = get(context, Feign.Builder.class)
        // required values
        .logger(logger)
        .encoder(get(context, Encoder.class))
        .decoder(get(context, Decoder.class))
        .contract(get(context, Contract.class));
    // @formatter:on
    configureFeign(context, builder);
    return builder;
}
```

## 

Feign.Builder 就是用来创建 Target 类的工厂类，这里会通过配置文件选取具体的工厂。





### get(FeignContext, Class) - 相关组件的获取





## FeignContext - 服务上下文集合

Feign 的实现中会为每个服务创建一个应用上下文，并且使用 FeignContext 保存，FeignContext 继承于 NamedContextFactory，作为应用中所有 Feign 上下文的集合。

> NamedContextFactory 中定义的是一组的应用上下文，并且应用上下文具有相同的配置类型，以及同一个父上下文。
>
> <img src="/home/chen/_note/pic/image-20210318223749982.png" alt="image-20210318223749982" style="zoom:67%;" />
>
> NameContextFactory 中 contexts 保存的就是所有的子上下文，configurations 保存的就是对应的配置类。



FeignContext 在 FeignAutoConfiguration 中被注册为Bean，并保存了所有的 FeignClientSpecification 类的 Bean 对象。

![image-20210318223958934](/home/chen/_note/pic/image-20210318223958934.png)





以下为创建 服务对应 ApplicationContext 的过程:

```java
// NamedContextFactory#createContext
protected AnnotationConfigApplicationContext createContext(String name) {
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
   if (this.configurations.containsKey(name)) {
      for (Class<?> configuration : this.configurations.get(name)
            .getConfiguration()) {
         context.register(configuration);
      }
   }
   for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
       // 这里寻找默认的配置项并注册
       // Debug 项目的是 RibbonConsulAutoConfiguration
      if (entry.getKey().startsWith("default.")) {
         for (Class<?> configuration : entry.getValue().getConfiguration()) {
            context.register(configuration);
         }
      }
   }
   context.register(PropertyPlaceholderAutoConfiguration.class,
         this.defaultConfigType);
   context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
         this.propertySourceName,
         Collections.<String, Object>singletonMap(this.propertyName, name)));
   if (this.parent != null) {
      // Uses Environment from parent as well as beans
      // 这里设置了父容器，使当前容器可以直接使用父容器的bean对象和环境
      context.setParent(this.parent);
      // jdk11 issue
      // https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
      context.setClassLoader(this.parent.getClassLoader());
   }
   context.setDisplayName(generateDisplayName(name));
   // 刷新，加载配置，加载Bean了，环境需要之前就设定好
   context.refresh();
   return context;
}
```

> FeignContext 创建的 ApplicationContext 非常简单，只有基础的 Feign 的配置类就开始 refresh，最终的 Bean 基本也就只有配置类中的那几个。

因为设定了父容器的关系，如果指定的 Bean 在配置类中灭有，也会进一步从父容器中获取。







FeignClientFactoryBean 创建过程中使用 get / getOptional 获取相关的组件和配置，如果子上下文找不到就会回到主上下文找。

FeignClientsConfiguration 是默认的配置，默认的配置并没有定义 Client，所以 Client 是从主上下文找的。



Feign.Builder 就是 Feign 的建造器类。

Targeter 就是在创建之前做一些额外的配置，DefaultTargeter 就是直接调用的 Feign.Builder#target 类，而 HystrixTargeter 则是继续配置 Fallback，FallbackFactory 以及 SetterFactory。

> Fallback 优先，配置了 Fallback 之后就不考虑 FallbackFactory 类。



HystrixFeign.Builder#builder 中增加了 invocationHandlerFactory 的属性，配置为 InvocationHandlerFactory。

Contract 用于解析注解，例如 RequestMapping 等就是通过 SpringMvcContract 解析出来的，开启 Hystrix 之后注册了 HystrixDelegatingContract，并且增加了 HystrixCommand 等的返回值判断。



Feign 会为每一个 FeignClient 类创建代理，InvocationHandlerFactory 就是创建代理的工厂，默认的代理就是

ReflectiveFeign.FeignInvocationHandler，如果使用 Hystrix 那么就会是HystrixInvocationHandler，工厂是在 HystrixFeign#build 时的内部类

每个







## 文章引用

[Feign终级解析](https://mp.weixin.qq.com/s?__biz=MzUwOTk1MTE5NQ==&mid=2247483724&idx=1&sn=03b5193f49920c1d286b56daff8b1a09&chksm=f90b2cf8ce7ca5ee6b56fb5e0ffa3176126ca3a68ba60fd8b9a3afd2fd1a2f8a201a2b765803&token=302932053&lang=zh_CN&scene=21#wechat_redirect)

