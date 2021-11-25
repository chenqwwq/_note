# AnnotationAwareAspectJAutoProxyCreator 

## 类图

![AnnotationAwareAspectJAutoProxyCreator](assets/AnnotationAwareAspectJAutoProxyCreator.png)

该类实现了 SmartInstantiationAwareBeanPostProcessor 接口，通过该接口实现了对容器中 Bean 对象创建过程的钩子。

实现了 ProxyProcessorSupport 类，可以用于创建代理对象，是 Aop 的基础设施类（AopIntrastructureBean）。

> Bean 的代理对象的创建逻辑主要在 AbstractAutoProxyCreator。

另外的实现了 BeanClassLoaderAware 以及 BeanFactoryAware，所以类中包含了当前所在的容器对象。

<br>

AnnotationAwareAspectJAutoProxyCreator 由 AspectJAutoProxyRegistrar 注入到当前容器中，AspectJAutoProxyRegistrar 由 @EnableAspectJAutoProxy 注解触发。

> AspectJAutoProxyRegistrar 主要的作用就是注册并配置 AnnotationAwareAspectJAutoProxyCreator。





## 概述

AbstractAutoProxyCreator 的有效实现包括如下几个方法：（按照可能的调用顺序

1. SmartInstantiationAwareBeanPostProcessor#predictBeanType(Class<?> beanClass, String beanName)
2. SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference(Object bean, String beanName)
3. InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
4. BeanPostProcessor#postProcessAfterInitialization(Object bean, String beanName)





### predictBeanType - 推断 Bean 的类型

```java
@Override
@Nullable
public Class<?> predictBeanType(Class<?> beanClass, String beanName) {
    // proxyTypes 保存的代理之后的类型
    if (this.proxyTypes.isEmpty()) {
        return null;
    }
    // 获取缓存的 Key，
    // 如果 beanName 不为空则以 beanName 作为 Key，如果是 FatcoryBean 则加上 &
    //  beanName 为空的时候以 beanClass 作为 Key
    Object cacheKey = getCacheKey(beanClass, beanName);
    // 从缓存中获取
    return this.proxyTypes.get(cacheKey);
}
```

proxyTypes 中保存的是对代理对象的映射，**该缓存在代理对象创建之后添加。**



<br>



### getEarlyBeanReference - 获取早期引用

![image-20211125155910262](assets/image-20211125155910262.png)

获取缓存的 Key，并且会**添加原始 bean 实例（未经过属性填充和初始化的对象）到 earlyProxyReferences** ，之后尝试代理该对象。





### wrapIfNecessary - 尝试创建代理对象

```java
//  AbstractAutoProxyCreator#wrapIfNecessary
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // targetSourcedBeans 保存的是自定义 TargetSource 创建之后填充的
    // 如果 targetSourcedBeans 存在该 Bean 表示已经创建过该代理
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    // 这里可以表明 advisedBeans 为 FALSE 的时候，是可以跳过创建
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    // 创建的 Class 是基础类型或者需要跳过当前创建流程
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        // 设置缓存，下次可以跳过
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    // 获取切到当前对象拦截器？
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }
	// 创建过一次之后就直接跳过了
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

首先是四类判断，判断对象是否可以被跳过。

第一类 targetSourceBeans，该缓存在自定义 TargetSource 创建之后添加。

第二类 advisedBeans 直接表明需要跳过本地代理创建流程。

第三类判断是否是基础类型（包括 Pointcut、Advisor、Advice、AopInfrastructureBean，因此可以通过继承该类跳过代理流程

第四类算是模板方法模式，可以让子类自定义其跳过代理的逻辑。

```java
// AspectJAwareAdvisorAutoProxyCreator#shouldSkip 
@Override
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
	// 获取候选的所有 Advisor 
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
    // 遍历所有的 Advisor
   for (Advisor advisor : candidateAdvisors) {
      if (advisor instanceof AspectJPointcutAdvisor &&
            ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
         return true;
      }
   }
   return super.shouldSkip(beanClass, beanName);
}
```

当前类是适配 AspectJPointcutAdvisor，如果此时创建的 Bean 对象被 AspectJPointcutAdvisor 切到就返回 TRUE，则直接跳过当前对象的代理创建。

之后在根据父级方法判断，父级方法直接调用的 AutoProxyUtils.isOriginalInstance(beanName, beanClass)。

![image-20211125162357325](assets/image-20211125162357325.png)

根据类名称判断，如果是以 ORIGINAL_INSTANCE_SUFFIX 为结尾的话就需要跳过。

> **在被 AspectJPointcutAdvisor 切到或者 BeanName 以 ORIGINAL_INSTANCE_SUFFIX 结尾的时候需要跳过。**

<br>

#### getAdvicesAndAdvisorsForBean - 获取合适的 Advice



