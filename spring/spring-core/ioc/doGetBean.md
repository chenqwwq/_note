# doGetBean 获取流程



## 概述

Bean 的创建



## 1. 解析 BeanName

去除 FactoryBean 的名称可能会携带的 & 符号。

逐层解析 Bean 对象的别名。



## 2. 尝试从缓存中获取

```java
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
        // 判断当前是否在创建中
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

> singletonObjects 存放的是已经创建完成的 Bean 对象
>
> earlySingletonObjects 表示早期创建的对象
>
> singletonFactories 
>
> singletonsCurrentlyInCreation 表示对象是否在创建中



### 判断是否在创建中

```java
public boolean isSingletonCurrentlyInCreation(String beanName) {
    return this.singletonsCurrentlyInCreation.contains(beanName);
}
```

通过 singletonsCurrentlyInCreation 表示 Bean是否在创建中。

![image-20210225111247663](/home/chen/github/_note/pic/image-20210225111247663.png)

以上两个方法其实包含在 getSingleton() 方法中，就是下图中的方法。

![image-20210225111411056](/home/chen/github/_note/pic/image-20210225111411056.png)



## 3. 解析获取到的早期 Bean

主要是判断 FactoryBean，如果是 FatcoryBean 则调用其 getObject() 方法。



#