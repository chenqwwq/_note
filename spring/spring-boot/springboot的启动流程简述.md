# Spring 相关启动流程综述

> SpringBoot 是对于 Spring 的进一步封装和扩展，并且在 Spring 的基础上对一些配置采取默认化，减少组装应用模块的时间。

---

[TOC]

---



## 概述

SpringBoot 的整个启动流程包含了环境（Environment）以及应用上下文（ApplicationContext）的准备。

启动过程中还会穿插有**监听器**和**应用初始化器**以及另外一些扩展点的执行。

<br>

## 相关组件

### Environment 应用环境

Environment 包含了整个 SpringBoot 中必要的环境配置以及参数，对整个应用作支持。

Environment 中包含了两类参数：Profiles 以及 PropertySources 。

Profiles 决定了当前应用处于某种环境，根据 Profiles 会加载到不同的 PropertySource。

> 实际的开发过程中，会根据开发和测试环境指定不同的 Profiles。

PropertySource 简单来说就是一个 K/V 对，对应的参数名以及参数值。

<br>

### ApplicationContext 应用上下文

**ApplicationContext 即是 BeanFactory 也是 EventPublisher，同时还是 ResourceLoader，**是整个 SpringBoot 框架核心中的核心。

ApplicationContext 作为 ApplicationEventPublisher 实现了相关的事件广播方法。

> 观察者模式或者说监听器模式的惯例，ApplicationContext 中需要持有所有监听器的引用。
>
> 这个是在 AbstractApplicationContext 中就实现的功能。

另外的作为 BeanFactory ，ApplicationContext 同时负责了 BeanDefinition 以及 Bean 对象的生命周期的管理。



### Listener 监听器

SpringBoot 中的 Listener 类的基类就是 ApplicationListener，该类型监听器响应的事件就是 ApplicationEvent。

> 这套监听器模式的组合是各自实现的 JDK 中的 EventObject 以及 EventListener。

另外的在 SpringBoot 的启动过程中还会出现 SpringApplicationRunListener，该接口一定程度上就是对启动过程的时期划分，以及每个时期可以做出的自定义扩展。

在深入一点，其实默认实现中，SpringApplicationRunListener 是 ApplicationContext 在初始化完成之前的 EventPublisher，代替 ApplicationContext 广播事件。

<img src="/home/chen/_note/pic/image-20210330224157350.png" alt="image-20210330224157350" style="zoom:67%;" />

> 按照 SpringApplicationRunListener 其实就能整理出整个 SpringBoot 启动的基本流程了。 

SpringBoot 提供了默认的 EventPublishingRunListener 实现类，该类收集在 ApplicationContext 准备完毕之前的监听器，并广播对应时期的事件。

> 监听器的获取是通过 SpringBoot 提供了工厂加载机制，在应用上下文初始化器基本同时加载。







## 启动流程

> 以 SpringApplication#run(Class,String[]) 为起点。 

### 1. 获取各类组件

**在 SpringApplication 的构造方法中，通过 SpringFactoriesLoader 获取所有指定的 ApplicationListener 以及 ApplicationContextInitializer。**

> ApplicationListener 和 ApplicationContextInitializer 都会深度介入应用的启动流程。
>
> 此时都还保存在 SpringApplication 中。

另外还会根据当前是否存在 DispatcherServlet 等关键类，推断当前应用类型。

<img src="/home/chen/_note/pic/image-20210330233647577.png" alt="image-20210330233647577" style="zoom:67%;" />



> 该阶段的主要意义就是获取 ApplicationListener 和 ApplicationContextInitializer。

### 2. Spring 应用启动

通过 SPI 机制获取到 SpringApplicationRunListener 的实现类，默认是就是上文提到的 EventPublishingRunListener。

> EventPublishingRunListener 创建的时候就会传入 SpringApplication，所以它也持有了第一步获取的所有 ApplicationListener。

然后立马广播 ApplicationStartingEvent，表示应用启动流程的开始。

<img src="/home/chen/_note/pic/image-20210330233627158.png" alt="image-20210330233627158" style="zoom:67%;" />

> ApplicationStartingEvent 此时只携带当前的 SpringApplication 以及程序入参，后续的所有事件也会携带这两个参数。



> 该阶段的主要意义就是封装获取到的 ApplicationListener类，监听器在启动过程中至关重要。

### 3. 创建并初始化环境类

<img src="/home/chen/_note/pic/image-20210330233557561.png" alt="image-20210330233557561" style="zoom:67%;" />

该阶段负责的就是 Environment 类的初始化，会将一系列的配置已经系统属性封装到该类中（不是全部）。

程序的入参会被封装为 ApplicationArguments 类，并传入到创建的 Environment 类中，另外的还有配置的 Profile 以及基础的 PropertySource。

此时到了 environmentPrepared 的时期，会广播 ApplicationEnvironmentPreparedEvent 事件，会额外携带刚创建好的 Environment 对象，所以监听该事件就可以前置的对 Environment 做修改。

响应该事件的重要的监听器类如下：

1. BootstrapApplicationListener

   该类会在触发时创建 SpringCloud 相关的 bootstrap 容器。

2. ConfigFileApplicationListener 

   **该类会在读取所有本地的配置文件中的配置内容。**



> 该阶段的主要意思就是准备好后续操作需要的环境，环境中包含了运行环境的各类属性和配置，为后续上下文的初始化提供方便。

### 4. 创建应用上下文

会根据当前的应用类型创建不同的应用上下文类。

<img src="/home/chen/_note/pic/image-20210330233538249.png" alt="image-20210330233538249" style="zoom:67%;" />



> 这里就是个创建的过程。

### 5. 准备应用上下文

<img src="/home/chen/_note/pic/image-20210331001423010.png" alt="image-20210331001423010" style="zoom:67%;" />

该阶段会简单的配置 Environment 到创建好的上下文对象中，也会事先注册 BeanNameGenerator 以及 ResourceLoader 等全局的工具类。

最重要的还是应用所有的上下文初始化器，初始化器中值得注意的有以下几个:

1. PropertySourceBootstrapConfiguration

   在 SpringCloud 的环境下，该类负责从配置中心进一步获取配置项。

之后还会广播 ApplicationContextInitializedEvent，额外携带创建好的 ApplicationContext 对象。

> 该阶段目前没有特殊的实现，但是是应用上下文创建完成之后除了上下文初始化器外另一个可以修改创建的上下文的方式。
>
> 可以在里面增加 primarySource 或者 source，来干预当前阶段接下来的操作。

<img src="/home/chen/_note/pic/image-20210331073220201.png" alt="image-20210331073220201" style="zoom:67%;" />

接下来除了注册一些基础的 Bean 对象之外，还会加载主要的 BeanDefinition，简单点就是应用的启动类。

最后广播 ApplicationPreparedEvent。

> 这里可以看一下广播方法的实现，因为 ApplicationContext 已经创建，此时会将 ApplicationListener 复制一份到 ApplicationContext中。



> 此阶段主要的作用就是准备的应用上下文，配置环境，添加默认的 Bean 对象，应用所有的上下文初始化器并且加载主配置类。

### 6. 刷新应用上下文

在应用上下文准备好之后就可以开始刷新流程了。

```java
// AbstractApplicationContext#refresh
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

ConfigurationClassPostProcessor 读取所有的 Bean 对象。