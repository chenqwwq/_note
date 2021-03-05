# Feign





---

[TOC]

---

## Feign 的自动化配置



## FeignClient Bean注册流程

FeignClient 注解的解析流程在于 FeignClientsRegistrar，该类在 EnableFeignClients 注解中被加入到容器中。

<img src="/home/chen/_note/pic/image-20210304213239994.png" alt="image-20210304213239994" style="zoom:67%;" />



在 FeignClientsRegistrar 继承了 ImportBeanDefinitionRegistrar。

> ImportBeanDefinitionRegistrar 类借由 ConfigurationClassPostProcessor，在上下文刷新阶段就会调用该接口的  registerBeanDefinitions 方法。

<img src="/home/chen/_note/pic/image-20210304213302218.png" alt="image-20210304213302218" style="zoom:67%;" />

首先会向容器注册一个默认的配置类。

> 配置项中包括了 Contract，Encoder，Decoder，ConversionService 等基础的配置项。
>
> 也可以为每个 FeignClient 指定一个单独的配置类。

之后的 registerFeignClients 方法的作用则是扫描 FeignClient 注解，并向容器注册。

```java
// FeignClientsRegistrar#registerFeignClients
public void registerFeignClients(AnnotationMetadata metadata,
                                 BeanDefinitionRegistry registry) {
    // 获取类扫描器，它的功能就是扫描整个包的所有Java类
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
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



以下是配置类注册的流程:

```java
// FeignClientsRegistrar#registerClientConfiguration
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
                                         Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
        .genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    // 这里注册的config可能为空
    builder.addConstructorArgValue(configuration);
    // 使用BeanDefinitionRegistry注册BeanDefinition
    registry.registerBeanDefinition(
        // 组合的名称类似:dev-consul-server.FeignClientSpecification
        name + "." + FeignClientSpecification.class.getSimpleName(),
        builder.getBeanDefinition());
}
```

> FeignClientSpecification 可以看做 name/configuration 的组合类。

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

到这里 FeignClient 的对象就生成好了，最终注入到容器的对象类型是 FeignClientFactoryBean。

### 小结

Feign 借由 ImportBeanDefinitionRegistrar 接口，在容器初始化阶段扫描并注册了 BeanDefinition。

相关的 BeanDefinition 包括默认的配置类，每个 FeignClient 对象，以及其上指明的配置类。

> 最重要的，FeignClient 最终扫描完成后注册的是 FeignClientFactoryBean 对象，这是一个 FactoryBean 对象。
>
> 所以最终的代理对象创建还需要继续看 FeignClientFactoryBean#getObject() 方法。

## FeignClient 代理对象创建流程

最后创建真实调用的 Bean 对象就是 FeignClientFactoryBean#getObject()。

<img src="/home/chen/_note/pic/image-20210304232239220.png" alt="image-20210304232239220" style="zoom:67%;" />

内部封装了一层，直接调用的 getTarget() 方法。

```java
// FeignClientFactoryBean#getTarget
<T> T getTarget() {
    // 从 ApplicationContext 中获取 FeignContext 类Bean对象
    FeignContext context = applicationContext.getBean(FeignContext.class);
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

Feign.Builder 就是用来创建 Target 类的工厂类，这里会通过配置文件选取具体的工厂。





### 各类组件的获取流程

> 先说结论， Feign 会为每个 FeignClient 创建 ApplicationContext，上下文中仅包含了其配置类，如果没有则为默认的配置类。

FeignClientFactoryBean 中可以通过 get / getOptional 获取相关的组件，以下是 get 方法的源码：

<img src="/home/chen/_note/pic/image-20210304233225735.png" alt="image-20210304233225735" style="zoom:67%;" />

get 方法会从 FeignContext 中获取该类的对象:

<img src="/home/chen/_note/pic/image-20210304233306625.png" alt="image-20210304233306625" style="zoom:67%;" />

再往下追就看到，最终获取的对象还是从 ApplicationContext 来的，但此时的 ApplicationContext 并不是最初的或者 bootstrap(SpringCloud的)。

<img src="/home/chen/_note/pic/image-20210304233415972.png" alt="image-20210304233415972" style="zoom:67%;" />

从 getContext 方法中可以看到，根据 name 也就是 contextId 获取不同的 ApplicationContext，如果不存在则创建。

> FeignContext 继承于 NamedContextFactory 可以看做是简单的 name 到 ApplicationContext 的映射，使用 FeignClient 的 name 属性做 Key，每个 FeignClient 都会有自己专属的 ApplicationContext。

以下为创建 ApplicationContext 的过程:

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



## 文章引用

[Feign终级解析](https://mp.weixin.qq.com/s?__biz=MzUwOTk1MTE5NQ==&mid=2247483724&idx=1&sn=03b5193f49920c1d286b56daff8b1a09&chksm=f90b2cf8ce7ca5ee6b56fb5e0ffa3176126ca3a68ba60fd8b9a3afd2fd1a2f8a201a2b765803&token=302932053&lang=zh_CN&scene=21#wechat_redirect)

