# springmvc

---

[TOC]



## SpringBoot 整合 SpringMVC 的启动过程

1. 在 ServletWebServerApplicationContext#onRefresh() 中调用了 createWebServer() 方法启动一个 Web 应用

![image-20210224233641447](/home/chen/github/_note/pic/image-20210224233641447.png)

2. 开始创建 WebServer

![image-20210224234636797](/home/chen/github/_note/pic/image-20210224234636797.png)

> 在获取 ServletWebServerFactory 的时候可以自己指定，默认是 TomcatServletWebServerFactory，可以自行修改定义相关Bean。
>
> 可以修改协议，增加特殊的连接器等。



getWebServerFactory 获取 ServletWebServerFactory

```java
protected ServletWebServerFactory getWebServerFactory() {
    // Use bean names so that we don't consider the hierarchy
    // 尝试从BeanFactory中获取,默认为TomcatServletWebServerFactory
    String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
    if (beanNames.length == 0) {
        throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to missing "
                                              + "ServletWebServerFactory bean.");
    }
    if (beanNames.length > 1) {
        throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to multiple "
                                              + "ServletWebServerFactory beans : " + StringUtils.arrayToCommaDelimitedString(beanNames));
    }
    
    return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}

```

