# SpringCloud

---

## 启动流程

在 spring.factories 中定义的 ApplicationListener 中定义的子类实现。

<img src="/home/chen/github/_note/pic/image-20210228231630933.png" alt="image-20210228231630933" style="zoom:67%;" />

BootstrapApplicationListener 监听了 ApplicationEnvironmentPreparedEvent 事件，在环境准备完成之后创建 SpringCloud 的上下文，同样也是 ConfigurableApplicationContext，默认上下文名称为 bootstrap。

新创建的 Context 作为原先 SpringBoot Context的上下文。

> SpringCloud 的容器作为 SpringBoot 容器的父容器。

