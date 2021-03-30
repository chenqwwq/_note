# Spring Transaction



---

[TOC]

---

## 概述

Spring 的事务本质上是对底层数据库的重封装。



## 相关组件

### TransactionManager - 事务管理器

该接口为标记接口，其直接子接口有 PlatformTransactionManager 和 ReactiveTransactionManager。

以 PlatformTransactionManager 为例子:

<img src="https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210316224438850.png" alt="image-20210316224438850" style="zoom:67%;" />

实现的方法如下：

|                          方法名称                          |   方法作用   |
| :--------------------------------------------------------: | :----------: |
| getTransaction(@Nullable TransactionDefinition definition) | 获取当前事务 |
|              commit(TransactionStatus status)              |   提交事务   |
|             rollback(TransactionStatus status)             |   回滚事务   |

每种不同的数据访问实现，都可以定义自己的事务管理器，例如使用 Hibernate 访问数据库，

则新建的是 HibernateTransactionManager，也就有了自己的一套创建，提交和回滚逻辑。





### TransactionDefinition - 事务定义

该类中定义了基本的事务属性，具体的属性如下：

<img src="https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210316224752132.png" alt="image-20210316224752132" style="zoom:67%;" />

事务的属性有如下几个：

|   属性名    |    属性含义    |
| :---------: | :------------: |
| Propagation | 事件的传播属性 |
|  Isolation  | 事务的隔离级别 |
|   Timeout   | 事务的超时事件 |
|  ReadOnly   | 是否为只读事务 |
|    name     |    事务名称    |



另外的 TransactionDefinition 中还定义了事务的隔离级别和传播级别。

> 事务隔离级别就是不同事务并发时后互相之间的可见度。

| 隔离级别                   | 含义                                                         |
| -------------------------- | ------------------------------------------------------------ |
| ISOLATION_DEFAULT          | 默认的隔离级别，使用的底层数据源默认的隔离级别               |
| ISOLATION_READ_UNCOMMITTED | READ_UNCOMMITTED级别，可能会出现脏读，幻读，不可重复         |
| ISOLATION_READ_COMMITTED   | READ_COMMITTED级别，避免了脏读和不可重复读                   |
| ISOLATION_REPEATABLE_READ  | REPEATABLE_READ级别，在 InnoDB 的存储引擎下，该级别能避免幻读 |
| ISOLATION_SERIALIZABLE     | SERIALIZABLE级别，串行化执行各个事务，也就不存在并发问题     |



> 事务的传播级别定义了事务控制的边界，在方法调用间决定事务的逻辑。

| 传播级别                  | 含义                                                         | 影响                                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | 默认级别，方法需要在事务中执行，当前没有则新建，有则加入     | 以单个事务执行，如果加入外层事务，不管内外的异常都会触发回滚 |
| PROPAGATION_SUPPORTS      | 支持事务执行，当前有事务就加入，没有就以非事务模式执行       | 同上                                                         |
| PROPAGATION_MANDATORY     | 强制性以事务模型执行，如果当前没有事务则抛异常               | 同上                                                         |
| PROPAGATION_REQUIRES_NEW  | 新事务执行，该方法必须以单独的事务执行，当前有事务就暂时挂起，新开事务执行 | 以多个事务执行，内外的异常不会互相印象，里层的异常不会触发外层的回滚 |
| PROPAGATION_NOT_SUPPORTED | 不支持事务模式执行，若当前存在事务则先挂起                   | 必须以非事务模式执行                                         |
| PROPAGATION_NEVER         | 不能以事务模式执行，若当前存在事务则抛出异常                 | 必须以非事务模式执行                                         |
| PROPAGATION_NESTED        | 以嵌套事务执行，如果当前存在事务则嵌套执行，如果没有事务则新建 | 如果加入当前事务，则外层事务的异常会触发内层的回滚，而内层的异常对外层无影响 |

> 异常需要抛出到方法外，如果方法内捕获了异常，也就不会触发回滚。



### TransactionStatus - 事务状态

<img src="https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210316232310738.png" alt="image-20210316232310738" style="zoom:67%;" />

记录了当前事务的状态。



|      状态      |        含义        |
| :------------: | :----------------: |
|  rollbackOnly  |  事务是否只能回滚  |
| newTransaction |    是否为新事务    |
|   completed    |    事务是否完成    |
|  hasSavepoint  | 是否存在 SavePoint |



## 事务的自动化配置

通过 @EnableTransactionManagement 引入的 TransactionManagementConfigurationSelector 来加载具体的配置类。

例如如果使用的 JDK 的动态代理，引入的就是 ProxyTransactionManagementConfiguration。
