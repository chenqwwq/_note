# Resume Point

> 北枳

## NETWORK

### HTTP/HTTPS

### TCP/UDP

### DNS

### RESTFUL

### GraphQL

<https://blog.csdn.net/a745233700/article/details/114824602>

## JAVA

### Collection

- HashMap原理

>  Hash，以 Hash 函数将对象散列到一定范围之内，以此作为下标定位标准，实现插入和获取接近O(1)的复杂度。

- 线程安全的map？

> ConcurrentHashMap

- HashMap多线程情况下扩容会出现什么问题？

> 1. 数据丢失
> 2. 环链

- CurrentHashMap说一下怎么保证的线程安全？

> CAS + synchronized
>
> 1.7 为 ReentrantLock，以多个桶为一段实现分段锁，相对于整个数组上锁来说是细化了锁的粒度。

- put的流程？

> 1. 懒初始，如果数组没创建则此时创建
> 2. 计算 Hash, hash << 16 & hash，扰动函数
> 3. 定位到桶，没有的话以value 作为头节点新增
> 4. 遍历桶，可能是链表或者红黑树，插入结束后看是否可以变为树 8升树 6退链表

- 说一下HashMap在JDK7和JDK8的区别？

> 1. 增加红黑树，降低桶遍历的整体事件复杂度
> 2. 头插法变为尾插法，避免环链

### Stream

1. 看你对JDK8的stream流有了解，那你说下？

> 常用，但是没细看过原理。

### VCS

### Spring Family

- 你项目中使用到了Spring cloud，能说一下使用到的组件，已经他的作用？

> Feign + Consul + Ribbon + Hystrix + Sentinel + Zuul
>
> Http 客户端，简化对接流程完美适配 Ribbon + Hystrix + Sentinel
>
> Consul 注册中心 + 配置中心
>
> Ribbon 服务获取 + 客户端负载均衡
>
> Hystrix 限流，熔断，并发控制(线程池和信号量控制)
>
> Sentinel 同 Hystrix

你是使用的哪个Spring的扩展点，进行跨线程的日志处理？（AOP环绕通知）

- 你对于spring的扩展点如何扩展的？

> SpringBoot 实现 AOP 的方式？
>
> 1. 自己定义一个 Advisor 注解或者直接自己创建一个
> 2. 自己定义 BeanPostProcessor，拦截 Bean 创建过程，创建代理对象，类似 Async，BPP 中定义切面逻辑
> 3. 扫描 BeanFactory，类似 FeignClient 的扫描

- AOP所使用的设计模式？

> 代理模式，观察者模式？

- spring生命周期

> Bean 的生命周期？
>
> 从创建 Bean 开始，查询缓存 SingleObjects，不在就先从父容器获取，没有就创建。
>
> 获取 BeanDefinition，创建依赖对象，，实例化 ，填充数据，初始化，注册 Disposable 回调。
>
> 实例化和初始化前后 BeanPostProcessor 调用实例化有工厂方法，创建方法，构造函数，空构造函数填充一些 Value 等数据调用初始化函数

- 动态代理的区别

> JDK 的动态代理是实现相同的接口继承同个 ProxyInstance，将对接口的方法调用转移到了子类。
>
> Cglib 使用的是字节码修改

- 环绕通知是什么？

> 方法调用前后

- Spring AOP是怎么出入的，也就是怎么完成的AOP代理？

> 以代理对象替换真实对象，对 真实对象的调用都转移到了对 代理对象的替换，代理对象在真实方法调用的前后添加钩子方法。

- **那你能具体讲一下怎么基于cglib进行的动态代理？**

> 不知

1. 你这个基于Spring扩展点进行跨线程操作日志记录怎么做的？

- 你对于Spring的话，是怎么进行二次定制的？

> 对各类扩展点进行扩展，最大的 BeanPostProcessor 直接插手 Bean 对象的创建过程，使用代理模式添加功能。
>
> 在 Springboot 中使用监听器监听的各种过程，还有 EnvironmentPostProcessor，还有 ApplicationInitializer，BeanFactoryPostProcessor
>
> 自动化配置的添加，可以使用 Import 注册自己的配置逻辑

- 看过哪部分的源码？

> AOP，IOC，SpringBoot 的启动流程，Async，事务的，Retry 的，Feign，服务注册的，SpringMvc 的

1. Spring bean的生命周期
2. SpringCloud的注册中心用的哪一个？
3. 他的注册流程是什么样的？

> Consul 
>
> 在 SpringMVC 启动完毕之后有个 WebStartedSuccess类似的事件，监听该事件注册。
>
> ServiceRegister

- 注册中心的负载均衡这么做的？

> 从注册中心获取到真实的地址之后，通过 Ribbon 来做负载均衡。

1. 你在项目中如何基于Spring的扩展点完成相关的业务难题的？
2. 你这个扩展点是在哪个阶段生效的？（你确定是这个阶段吗?那你在我电脑里面找一下这个相应的接口在哪吧？我人晕了直接）
3. 在某一个utils类中（这个类没有交给Spring进行管理），我想要获取到Spring容器的某一个bean你该怎么做？

> 声明一个 ApplicationContextHolder 的工具类继承 ApplicatioContextAware。

1. 那Spring是什么时候或者是什么阶段调用的ApplicationContextAware接口的方法的？

> 是初始化阶段最先调用。

1. IOC的理解？
2. Spring容器启动流程？

> SpringBoot 的启动流程：
>
> 1. 创建环境上下文
>2. 配置环境上下文，此时触发 FileConfigListener
> 3. 创建 ApplicationContext
>4. 准备 ApplicationContext 
>    - 此时可能顺带触发 SpringCloud 的Bootsrtap，先创建SpringCloud的上下文，并作为当前上下文的父上下文，SpringCloud 的
>   - 还有PropertySourceBootstrapConfiguration，用于配置中心的配置数据加载
>    - 还会加载所有的BeanDefinition
>   - 对 Import 和 ImportSelect 的处理
> 5. 刷新上下文
>
>    1. 创建并准备 BeanFactory
>    2. 执行 BeanFatocyrPostProcessor
>    3. 加载 BeanPostProcessor
>    4. 
>
>    

1. BeanFactory和ApplicationContext有什么区别？

> ApplicationContext 是对 BeanFactory 的扩展，完全包含了 BeanFactory 的功能之外还提供了Listener的支持，以及对各类环境的配置和整合。

1. Spring如何解决循环依赖问题

> 三级缓存，缓存已经构建完成的Bean,SingleObjects，缓存刚实例化好但是并没有填充数据的对象，以及 FactoryBean 对象。

- Spring框架中的Bean是线程安全的么？如果线程不安全，那么如何处理？

> Bean 默认为单例模式，线程不拿全

- Spring事务的实现方式和实现原理

> Spring 的事务基于 AOP 开发，定义了一个 TransactionManageer 来实现各种的事务操作，包括挂起，回复，开启，提交，回滚等。
>
> 使用 ThreadLocal 保存当前的事务名称，事务状态，所以当开启一个事务的时候会先判断线程本地是否有为提交的事务，有的话根据隔离级别执行隔离逻辑。

1. <https://blog.csdn.net/a745233700/article/details/80959716>
2. <https://blog.csdn.net/oldshaui/article/details/90675149>（spring cloud）
3. <https://www.cnblogs.com/wjqhuaxia/p/11837069.html>（同上）
4. <https://blog.csdn.net/qq_35906921/article/details/84032874>（同上）

### MyBatis/MyBatis-plus

1. MyBatis的一级缓存和二级缓存说一下？使用场景？
2. 两个框架的区别？plus做了什么
3. plus的原理？
4. MyBatis是如何进行分页的？分页插件的原理是什么？
5. <https://blog.csdn.net/a745233700/article/details/80977133>

### WebFlux

1. <https://zhuanlan.zhihu.com/p/92460075>

### JVM/JMM在

- JVM运行时数据区，存放的内容

> 堆，JVM 栈，本地方法栈，元空间，PC
>
> 堆中主要存放的是Java的对象
>
> JVM栈是线程私有的，栈帧里面存放的有局部变量表，操作数栈，应用，方法返回地址

- JAVA内存模型？（JMM内存模型说一下吧？）

>  JMM 是虚拟机规范中定义的一种模型，使得虚拟机可以屏蔽顶层的硬件区别。
>
> JMM 中规定数据保存在主内存，操作时加载到线程本地，通过JMM规范进行同步，相当于说线程之间必须要通过主内存通信。
>
> Happen-before原则以及七种绝对安全操作，load,use,assign,lock,unlock,read,write

- 类加载机制？

> 加载，验证，准备，解析，初始化

- 为什么要这样加载类？

> 加载负责的是找到 Class文件的二进制流，并转化为Class对象的数据结构
>
> 连接过程是保护Class二进制流的安全性，以及固定内存的分配和连接的转换

1. 常见的垃圾收集器？

> CMS,G1,Serial,Seral Old,pararrl ，parall old

1. CMS和G1的区别？

> CMS 是老年代垃圾收集器，默认和par New 结合使用，G1不区分老年代和新生代，将整个堆空间划分为 region，region 有老年型和新生型，还有Huge类性
>
> CMS 主要是降低用户的延迟，但是G1可以达到一个可预测的停顿时间，对于region的整理预处理后会有一个排序，尽量收集收益较高的区域

1. JVM参数你是怎么设置的

> xms，xmx 设置为一样的。
>
> 整个堆空间扩展的时候，也会增加用户进程的停顿。
>
> 添加 GC 日志以及OOM时的Dump文件。

### JUC/Current Class（TL ed.）/AQS/Lock/keyword

1. 你项目中使用多线程编程的场景？要突出在项目中如何使用的？
2. reentrantlock和synchronized的区别
3. 线程池使用过没？线程池的参数？(数量公式)
4. 你说你用了ThreadLocal，说说他的作用？实现原理？
5. synchronize的原理说一下？
6. 在多线程中，如何让一个线程不再进行执行了，中断他的执行？
7. 多线程并发在项目中用过哪些？
8. ThreadLocal的话，不显示调用remove方法的话，除了内存遗漏问题的话，还会有什么其他的问题？
9. 那如果出现这种问题，你大概知道是因为这个线程池中线程没有进行remove，那你可以使用什么办法确定你的猜想？
10. AQS说一下原理

> AQS 和 JVM 原生的 Monitor 机制类似，使用了变种的CLH同步队列来解决线程的排队和饥饿问题。
>
> 本身维护了一个双端队列作为阻塞队列，获取失败线程入队列，每个节点通过监听前驱节点的状态来判断是否可以竞争锁，自旋一定次数未获取到锁则使用LockSupport挂起当前线程。
>
> 在队首节点执行完成之后，会唤醒后继的节点，如果是共享模式会一直往后知道遇到一个独占节点，另外还有取消等节点状态。
>
> Condition 是一个单向队列，存放的是因为条件不满足的阻塞节点，如果条件满足之后会从 条件队列加入到阻塞队列，可以存在多个单向队列。

1. 说一下volatile关键字吧？

> Volatile 是相对轻量级的同步关键字。
>
> 保证对象的可见行和有序性。

### Other

1. 类加载器有哪些，一个类加载器如何加载类的（类加载机制）

> Bootsrtap,Ext,App 三种主要的加载器。

1. 为什么要使用这样的加载机制加载类，不使用行不行？
2. tomcat是使用这种类加载机制吗？

> Tomcat 部分支持该种累加在机制，。

1. String字符串的最大长度是多少？
2. IO模型都有哪些？
3. wait、notify和await、signal两套之间的区别？
4. 使用ReentrantLock时为什么signal就行，而使用Object.notifyAll是通知全部的？
5. 有哪些JDK的核心类库用到了阻塞队列这种数据结构？
6. 类加载的过程？ 每一步都是干什么的？
7. 你项目中每一种规则都对应一个线程，而且线程会从阻塞队列中获取GPS信息，那么的话，你这如果线程挂了（假设他挂了，不管怎么挂了），那么的话，你这阻塞队列中满了或者是超过了最大的长度，那么该如何处理呢？
8. 看你对JDK8的stream流有了解，那你说一下JDK7和JDK8的区别，或者说有哪些优化？

## Open Source Project

### Redission

1. 实现分布式锁的原理

### Lecture

### Guava Event Bus

### Guava cache/Caffeine

### Quartz/xxl Job

1. 你项目中定时任务是怎么处理的？xxl Job公司内部开发的，他的运行流程？

### ~~ELS~~

###  ~~Netty~~

### ~~Dubbo~~

1. dubbo用过没？

## DATASOURCE

### 标准 SQL语句

1.  SQL语句的编写，成绩表，主要字段：学生id、课程id、成绩， 使用SQL语句编写   查询出每个学生的平均成绩大于70的学生的id，和平均的分数

### PostgreSQL

### MySQL

1. MySQL和PG的区别？
2. 那他们之间有使用区别呢？
3. MySQL的优化？

> Explain 
>
> 

1. 常用的存储引擎？InnoDB与MyISAM的区别？
2. 事务的ACID与实现原理？

> 原子性，一致性，隔离性，持久性
>
> 原子性通过 undolog 实现，如果异常则执行undolog回滚操作
>
> 隔离性通过锁和mvcc实现，根据不同的隔离级别有不同的实现。
>
> 持久性通过 redolog + binlog实现

### 索引原理/B+ Tree

1. 怎么建立索引，建立索引的原则是什么？
2. 对于建立索引的是，应该怎么建立索引？
3. 在哪些字段上建立索引？
4. 一个表中建立索引的个数限制？  
5. 对于大字段建立索引的长度？
6. MySQL索引的实现原理

### 分库分表

1. GPS的分表规则是什么？
2. 整个分表是怎么写的，是自己写的，还是使用的框架？
3. PostgreSQL做分表处理的时候，和MySQL做分表有什么区别？
4. 那分库分表的中间件有哪些？
5. sharding-jdbc的分库规则有哪些？
6. 怎么建立索引，建立索引的原则是什么？
7. 分库分表：**垂直分表、垂直分库、水平分表、水平分库**
8. 分区

### 优化/优化步骤

1. 跨多表查询是怎么处理，怎么解决查询性能？
2. 查询近一个月的某一个车辆是否是违规行为，那你岂不是要联合30个表进行查询，那你是怎么处理？
3. 查询的性能优化，如果不进行性能优化的话，那查询效率会很慢的啊，因为要关联30个表，你是怎么做的？
4. 如果这个字段不是id，而且一个普通字段，那么该怎么查询？
5. 一般优化步骤有哪些（explain）
6. 查询的优化，遇到查询慢该怎么解决？
7. 如果加了所有，查询也走了索引，可是查询还是慢，那是怎么导致的？ 怎么解决？
8. SQL优化和索引优化、表结构优化
9. 数据库参数优化

### 主从/集群

1. 我们是使用主从读写分离的操作，如果我操作主节点之后，在操作从节点时，想要立即看到刚才操作的数据，应该怎么做？
2. 数据库主从的架构是什么样的？
3. 数据库主的宕机后，那你修改怎么整的？ 
4. 数据库主从部署在主节点操作数据以后，其实其他的去查询从，如果我想看到最新的数据怎么办？
5. 那如果从节点很多，那你这速度太慢怎么办？
6. MySQL的主从复制
7. 读写分离

### Transaction/MVCC

- 说一下MVCC是什么?

> MVCC 多版本并发控制，是 InnoDB 的并发控制策略，在系统中维护快照，使读不上锁，

- 间隙锁的话解决了什么问题，和MVCC有冲突吗？

> 间隙锁解决的是幻读的问题，防止在事务过程中插入，没有冲突于

- RR和RC级别下面是，事务里面的ReadView视图有什么区别呢？

> RR 级别下只会在第一次 Select 的时候创建快照，RC 级别下每次 SELECT 都会创建快照。

- 在普通的select（不加锁的读）为什么RR可以解决幻读，而RC不能解决幻读呢？以ReadView作为出发点进行思考？

> 

1. 你是怎么使用RabbitMQ保证的分布式事务
2. RabbitMQ保证分布式事物在项目的使用场景，为什么要用？不用行不行？
3. 分布式事务？
4. 事务的ACID与实现原理？
5. 数据库中的锁机制？

### Other

1. 你在设计表时，需要考虑的原因是什么?
2. 主键一般用自增ID还是UUID？
3. <https://blog.csdn.net/a745233700/article/details/114242960>

## OS

### LINUX

1. 常见指令

## Other

1. 用户态和内核态，为什么需要切换？
2. <https://blog.csdn.net/a745233700/article/details/85995095>

## 中间件

### Redis

1. 你项目使用Redis的场景，使用到了哪些数据结构？
2. Redis你在项目中怎么使用的？用在哪个地方？解决了什么问题？
3. 那你怎么保证Redis和数据库的数据一致性？
4. 那如果是Redis先更新数据库成功，删除Redis失败了？（先更新数据库，再删除缓存会有什么问题？）
5. 如果是金融行业呢？对Redis要求比较高，那你怎么办？（如果对于一致性要求高怎么处理？）
6. 延迟双删也可能有问题啊？(看下文章，三种解决方案)
7. Redis他的数据类型和底层数据结构的实现？
8. **有没有想过Redis中Zset是使用跳表进行实现的？那他为什么不使用像数据库那种的树形结构进行实现？**
9. Redis为什么这么快？
10. Redis的使用场景有哪些？就是Redis有哪些用途？
11. 布隆过滤器的实现原理是什么？
12. 那你使用Redis是用来干嘛的呢？
13. Redis 有哪些架构模式？讲讲各自的特点
14. 什么是Redis持久化？Redis有哪几种持久化方式？优缺点是什么？
15. 刚刚上面你有提到Redis 通讯协议(RESP )，能解释下什么是RESP？有什么特点？
16. 什么是一致性哈希算法？什么是哈希槽？
17. 使用过Redis分布式锁么，它是怎么实现的？
18. 使用过Redis做异步队列么，你是怎么用的？有什么缺点？（一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。缺点：在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如RabbitMQ等）
19. 能不能生产一次消费多次呢？（使用pub/sub主题订阅者模式，可以实现1:N的消息队列。）
20. 什么是缓存穿透？如何避免？什么是缓存雪崩？何如避免？
21. <https://zhuanlan.zhihu.com/p/91539644>

### Nginx

1. Nginx负载均衡算法有哪些？
2. 反向代理（反向代理服务器可以隐藏源服务器的存在和特征。它充当互联网云和web服务器之间的中间层。这对于安全方面来说是很好的，特别是当您使用web托管服务时。）
3. Nginx用途（`Nginx`服务器的最佳用法是在网络上部署动态`HTTP`内容，使用`SCGI`、`WSGI`应用程序服务器、用于脚本的`FastCGI`处理程序。它还可以作为负载均衡器。）

### RabbitMQ

1. 你是怎么使用RabbitMQ保证的分布式事务
2. RabbitMQ保证分布式事物在项目的使用场景，为什么要用？不用行不行？
3. 为什么使用RabbitMQ来接收GPS信息？为什么不用Kafka接收呢? Kafka吞吐量还高？
4. MQ消息丢失问题怎么处理？
5. 如要要保证消息不丢失呢？
6. 和Kafka，RocketMQ进行了对比吗？
7. 如何保证消息不被重复消费？
8. 有序性？
9. 消息堆积？
10. 保证高可用？
11. https://blog.csdn.net/a745233700/article/details/115060109

### ~~Kafka~~

1. Kafka也有路由健的选择的，这个好像不是原因吧？（你感觉你的项目还有哪些可以改进的点？）
2. Kafka有没有了解？（没有）

### ActiveMQ

1. 对比其他MQ说一下？
2. 说下原理？
3. 用到的协议和其他有什么不同？AMQP

### Zookeeper

1. 你说你使用到了zookeeper解决这个问题，那么你有没有对其他的技术方案进行选择，还是说对于自己的认知，就选择了zookeeper？（保持又服务器宕机时的通知，和RabbitMQ一致，保证容灾性）
2. zookeeper的zab你说一下吧？
3. <https://www.cnblogs.com/bigband/p/13574344.html>

### Tomcat

1. tomcat是使用这种类加载机制吗？
2. Tomcat的优化点
3. 怎么设置参数，输出文件的路径

## Data structure/Algorithm

### Application Algorithm

1. 你这个一致性hash怎么用的？（保证数据上传的有序性）
2. 一致性hash还用哪些应用场景？

### Base Algorithm

1. 删除排序链表中的重复元素
2. 删除排序链表中的重复元素 II
3. 阻塞队列的实现，生产者和消费者模式
4. 除了ReentrantLock还有其他的什么方式可以实现？

## Server Problem

### OOM

1. 线上有没有遇到过OOM问题？怎么解决的？（两种OOM，看阿里开源工具文章）
2. 应该怎么避免OOM问题？

### IO

1. IO超载的排查、解决方法
2. 磁盘运行原理

### CPU

1. CPU经常打满了，怎么优化？
2. 运行原理

### 宕机

1. 如果MQ宕机了怎么办？
2. 如果数量太大，导致服务器宕机了，那你该怎么处理？
3. 对于服务器宕机的问题，如果我想不影响数据的迁移怎么整？
4. 数据库主的宕机后，那你修改怎么整的？

### Other

1. 如果前方技术支持反映系统运行比较慢，那么你该如何处理（系统反应慢的处理）？
2. 线上服务器反应慢，你该怎么办？

## OTHER

1. 对于你业务中，围栏分析服务器集群中，可能存在服务器他的硬件设备不同，导致处理消费的能力也不同，那如果我想让每台服务器服务的消息数差不多，也就是每一台服务器都让他每秒消费60秒，做到均匀消费，那你该怎么样？
2. 数据转换，从数据产生到最后处理的流程说一下？
3. 接受到的数据是结构化的数据吗？
4. 这样的话，你之前的解决问题的方案是不是就不需要这么麻烦了，还需要每个服务器一个MQ？
5. 要保证数据的有序性，你把同一个设备的MQ发到不同的设备，那你不知道哪个信息会先被处理，还是会有问题的
6. 那我只使用同一个MQ，因为是按生产者存放的顺序整的，这样是不是也能保证有序性，不用这么麻烦？
7. 那你只使用到一个MQ队列，那他的吞吐量怎么保证呢？
8. 如果业务继续扩大呢？那你应该怎么处理？
9. 消息从产生到最终的数据库存储，它中间的处理流程是怎么样的？
10. 对于判断一个GPS经纬度是否出区域，你们是怎么判断的？(不清楚)
11. 那你说一下undo和redo log是什么？分别解决了什么问题？
12. 你这每次服务器宕机和新服务器加入都需要重新进行订阅，那么如果你的设备有1000万，那你这传递的设备id将会是很多的，怎么解决？
13. <https://blog.csdn.net/a745233700/article/details/114006686>