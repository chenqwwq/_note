# AOP相关类



## Pointcut

![image-20210131204622387](/home/chen/github/_note/pic/image-20210131204622387.png)



**切点接口，该接口用来匹配切点，在切点上继续执行Advice**

匹配的方式有两种，`ClassFilter`和`MethodMatcher`，前者用来匹配类，后者匹配具体方法。



Spring中的常用实现有以下几种:

1. `AnnotationMatchingPointcut`

该类是最常用的几种之一，用来匹配某个注解，可以在类上也可以在方法上。

![image-20210131214807937](/home/chen/github/_note/pic/image-20210131214807937.png)

以上的构造函数可以看到，类和方法的匹配分别是`AnnotationClassFilter`以及`AnnotationMethodMatcher`。

指定的Pointcut用来在生成代理类时判断是否可以被代理。

2. NameMatchMethodPointcut

按照方法的名称来匹配，ClassFilter固定为ClassFilter.TRUE。







## Advice

通知类，该类在Spring的包中为一个空接口。

简单来说，每一个Advice都是一个在Pointcut切中目标时需要执行的方法。



Advice可以分为以下两种:

1. 基础的Advice
   - AfterReturningAdvice
   - ThrowsAdvice
   - MethodBeforeAdvice
2. 基于AspectJ的Advice
   - AspectJMethodBeforeAdvice -  继承了MethodBeforeAdvice，解析`org.aspectj.lang.annotation.Before`，下同
   - AspectJAfterAdvice  - `@After`
   - AspectJAfterThrowingAdvice  -  `@AfterThrowing`
   - AspectJAfterReturningAdvice  -  `@AfterReturninf`
   - AspectJAroundAdvice - `@Around`



但是在Proxy中最终都会被`AdvisorAdapter`适配为`MethodInterceptor`，这里是用了适配器模式。

![image-20210131222356539](/home/chen/github/_note/pic/image-20210131222356539.png)

**`Interceptor/MethodInterceptor`**的固定实现:

- `ExposeInvocationInterceptor`

  该类用于在执行前将需要调用的方法解析出来并保存到一个ThreadLocal中，作为缓存。

- AfterReturningAdviceInterceptor/MethodBeforeAdviceInterceptor/MethodBeforeAdviceInterceptor

  对于普通的Advice的适配方法，例如AfterReturningsAdvice在适配后就变成了AfterReturnningAdviceAinterceptor

- AbstractTraceInterceptor

  抽象的日志类，子类都是负责相关的日志记录，以下是整体的调用方法:

  ![image-20210131231856989](/home/chen/github/_note/pic/image-20210131231856989.png)

  具体的方法调用，invokeUnderTrace为模板方法在子类中实现。

  粗略的继承者有以下几个。

  ​	                                   ![image-20210131231746122](/home/chen/github/_note/pic/image-20210131231746122.png)	

  CustomizableTraceInterceptor会在方法调用的前后打日志。

  





## Advisor

切面类，该类就是Pointcut和Advice的组合，也就包含了一组需要执行的方法以及需要执行点。

常用的如下:

1. PointcutAdvisor

   该类组合了Pointcut和Advice两种工具类，包含了两种工具类的get方法。

2. AspectJPointcutAdvisor

   该类用于对AOP的命名空间的解析。

3. InstantiationModelAwarePointcutAdvisorImpl

   该类用来表示被@AspectJ标注的类的解析结果。

   在`AnnotationAwareAspectJAutoProxyCreator`中，会一次性获取所有的Advisor相关类(直接继承Advisor以及标注@AspectJ)，上层解析方法如下:

   ![image-20210131224056857](/home/chen/github/_note/pic/image-20210131224056857.png)

   上图方法中，`super.findCandidateAdvisors()`会检索出所有的`Advisor`的自雷，而`BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors`则会找出所有的`@AspectJ`。

   4. NameMatchMethodPointcutAdvisor

      该类是指定使用`NameMatchMethodPointcut`切面类，不过Advice需要自定义。





## Advised 

该类并不是完全明白，但简单的可以当做是代理对象的创建者。

![image-20210131232148813](/home/chen/github/_note/pic/image-20210131232148813.png)

注释表明，该接口的子类持有AOP代理类的配置，该配置包括了拦截器以及切点以及被代理的接口。

Spring中的任何代理类都可以从该接口中获取。(我也不知道翻译的什么东西)



总共有以下三种实现方式：

1. ProxyFactory 
2. ProxyFactoryBean
3. AspectJProxyFactory - 用来处理@AspectJ

都是创建代理类的主要方法，以ProxyFactory举例，创建的方法简单如下:

![image-20210131234456371](/home/chen/github/_note/pic/image-20210131234456371.png)





## 中间总结

> 从以上内容来看，在Spring中使用AOP主要就是定义Advisor，通过继承PointcutAdvisor或者使用@AspectJ注解。
>
> 进一步来看就是定义Pointcut，该类用来确定切点，还有就是Advice，该类用来表示切点该执行哪些方法。
>
> Advice可能是早期的接口，所以后续的实现会使用适配器模式将Advice转换为Interceptor，来实现对方法的拦截。



> 简单梳理以下代理类的创建:
>
> 1. 通过BeanPostProcessor介入Bean的创建流程，例如AnnotationAwareAspectJAutoProxyCreator
>
> 2. 对代理类的创建流程框架主要还是在AbstractAutoProxyCreator中，存在以下三种情况的介入:
>
>    1. SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference
>
>       该方法用于提前暴露的Bean对象的获取
>
>    2. InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation 
>
>       在实例化前如果该方法成功获取具体类，就不会走到`createBean`的流程
>
>    3. BeanPostProcessor#postProcessAfterInitialization
>
>       该方法在初始化后进行，对已经创建好并初始化完毕的实例进行操作
>
>    ​    以上三种方法的创建流程基本一样，重点在`wrapIfNecessary`。
>
> 3. 直接返回缓存的类，过滤不需要代理的类
>
> 4. 获取对象可以使用的Advisor
>
> 5. 通过ProxyFactory进行创建
>
> 6. ProxyFactory中有JdkDynamicAopProxy和CglibAopProxy两种创建方式，这两个类的`getProxy`才是具体创建的方法。



> 通常情况下，所有的Object类的方法都不会被Spring的AOP代理。
>
> 在5.x版本的Spring中除了纯接口，类似CurdRep这种，其他的都是以CGLIB形式进行代理。



## AdvisedSupport

存放AOP代理的所有配置信息，所有支持的Advisor，包装的TargetSource，以及AdvisorChainFactory。





## AopProxy

![image-20210201001740754](/home/chen/github/_note/pic/image-20210201001740754.png)

对象创建的顶级接口，具体实现主要有`JdkDynamicAopProxy`以及`CglibAopProxy`两个。



## BeanFactoryAspectJAdvisorsBuilder

根据`@AspectJ`注解生成Advisor类。

![image-20210131212007062](/home/chen/github/_note/pic/image-20210131212007062.png)

注释的意思是帮助检索`@AspectJ`注解标注的类，以及生成Spring Advisor。



## BeanFactoryAdvisorRetrievalHelper

找到工作目录下的所有`Advisor`类。

```java
public List<Advisor> findAdvisorBeans() {
    // Determine list of advisor bean names, if not cached already.
    // 缓存的Advisor名称
    String[] advisorNames = this.cachedAdvisorBeanNames;
    if (advisorNames == null) {
        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the auto-proxy creator apply to them!
        // 获取所有的Advisor的Bean名称，此时不需要初始化
        advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
        this.cachedAdvisorBeanNames = advisorNames;
    }
    if (advisorNames.length == 0) {
        return new ArrayList<>();
    }

    List<Advisor> advisors = new ArrayList<>();
    // 逐个初始化Eligible的Advisor
    for (String name : advisorNames) {
        if (isEligibleBean(name)) {
            if (this.beanFactory.isCurrentlyInCreation(name)) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Skipping currently created advisor '" + name + "'");
                }
            }
            else {
                try {
                    advisors.add(this.beanFactory.getBean(name, Advisor.class));
                }
                catch (BeanCreationException ex) {
                    Throwable rootCause = ex.getMostSpecificCause();
                    if (rootCause instanceof BeanCurrentlyInCreationException) {
                        BeanCreationException bce = (BeanCreationException) rootCause;
                        String bceBeanName = bce.getBeanName();
                        if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                            if (logger.isTraceEnabled()) {
                                logger.trace("Skipping advisor '" + name +
                                             "' with dependency on currently created bean: " + ex.getMessage());
                            }
                            // Ignore: indicates a reference back to the bean we're trying to advise.
                            // We want to find advisors other than the currently created bean itself.
                            continue;
                        }
                    }
                    throw ex;
                }
            }
        }
    }
    return advisors;
}
```

该类就是在BeanFactory中将所有继承Advisor的子类全部找出来。





> `BeanFactoryAspectJAdvisorsBuilder`和`BeanFactoryAdvisorRetrievalHelper`都是用来查找`Advisor`类的。



##  TargetSource

AOP主要使用代理模式，增强被代理的功能，但不是直接代理的目标类，目标类会被包装成一个TargetSource类，是代理类和被代理类之间的解耦，可以实现类似多个实例公用单个代理类的功能。

Spring默认提供了以下默认实现:

1. SingletonTargetSource - 单个对象的代理
2. AbstractPoolingTargetSource - 抽象对象池的代理，可以实现上述说的多个实例共享单个代理类的功能，类似的采用`apache.commons.pool2`中的对象池实现。





### AnnotationAwareAspectJAutoProxyCreator

最终的代理自动创建类。