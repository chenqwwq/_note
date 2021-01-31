## 以Consul作为配置中心



1. PropertySourceLocator

   用于定位具体的资源位置，每个配置中心都可以继承该类，来获取自己的PropertySourceLocator

2. ConsulPropertySourceLocator

   consul-config用来定位consul中的配置信息的实现类，使用的目录如下：

   - ${spring.cloud.config.consul.prefix}/application
   - ${spring.cloud.config.consul.prefix}/${spring.application.name}

   以上两种再加上分别遍历profiles的结果类似:

   ${spring.cloud.config.consul.prefix}/application,dev/

3. PropertySourceBootstrapConfiguration

   **继承ApplicationContextInitializer，在SpringBoot的启动过程中准备ApplicationContext的时候调用，通过SPI的方式引入。**





> ConsulPropertySourceLocator最后会被包装成一个BootstrapPropertySource。
>
> 通用名称为`bootsrtapProperties-{data-dir}`，eg: `bootstrapProperties-config/application`

> consul的配置地址默认会有一个`${prefix}/application/${data-key}`

> Enviroment继承了PropertyResolver，所以可以使用类似`environment.resolvePlaceholders("${logging.config:}")`的方式直接获得占位符的实际值。

> 在SpringBoot启动的最后阶段，`finishRefresh`中，会初始化`LifecycleProcessor`，并调用其中的`onRefresh`方法。