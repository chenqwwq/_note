> Spring中bean的生命周期

Spring 中 Bean 的创建流程：

1. 解析出 BeanDefinition 需要合并父级
2. 处理 OverrideMethod 包括 @LookUp 注解
3. BPP#BeforeInstant 前置实例化钩子，成功之后走后置初始化流程 BPP@AfterInit （ 这部分是自定义的创建 Bean 流程，失败之后开始 Spring 定义的实例化流程，总共可以分为实例化，填充属性，初始化三个流程
4. 实例化 ，Spring 提供 Suppier ，FactoryMethod（@Bean 标注在方法上），构造函数（构造函数可以通过 BPP 选择
5. 实力化之后调用 SmartBPP#Merge（只提供 BeanDefintion，不包含刚创建的 Bean 对象
6. 满足要求暴露早起引用 exposeEarlyReference
7. 接下来是填充属性 - 先调用 BPP#AfterInstan
8. 推断 AutowiredMode（好像用到不多了，这里是直接塞进去的
9. BPP#PropertyValues，BPP#Properties 收集属性并在最后注入（包括 Resource 所在的 ComonAnnotation 和 Autowire 所在的 AutowireAnnotation
10. 接下来是初始化 - 调用部分 Aware 接口（还有部分在 BPP 实现
11. BPP#BeforeInit
12. init-method IniBean
13. BPP#AfterInit
14. 注册回收的生命周期 DispoaBean



> 为什么说反射会影响性能 有去研究过为什么影响性能吗？

Java 的反射基本是基于 Unsafe 实现的，为了安全性 Unsafe 跳过了 JVM 层级的优化



>  静态代理和动态代理的区别

静态代理每次都需要重新编写代理对象，动态代理可以在 Runtime 期间生成代理对象。

JDK 动态代理需要继承接口并且实现 InnovcationHandler，最后生成的代理对象，实现传入的接口并且持有 IH 对象，所有的方法调用都会传入 IH。



> jdk代理和cglib代理的区别 性能会高一些

JDK 动态代理类似于接口实现，要求代理的对象一定要实现接口，而 cglib 对象更接近继承，要求被代理的对象必须可继承（不能用 final，包括方法





> 5.Sringboot的自动装配

@EnableXXX 带着 @Import 原注解，会在 ConfigClass 处理时提取出 ImportSelect 并加载 方法返回的对象。

另外还配上 Spring 的工厂加载模式，SpringFactory@load 类似的静态调用会搜索 spring.factories 并返回 Enable 下的所有配置累，类似的有 @EnableAutoConfiguration

条件注解，@ConditionOnClass 等。



> 如何解决循环依赖

三重缓存，分别存放已经创建好的 Bean 对象，getEarlyReference 包裹的 ObjectFactory 调用，还有经过代理但是没有初始化的对象。

多一层缓存实现对象的懒初始化



> 注解注入和构造器注入

？？？



> SPring注入有那些方式

构造器注入（注入之间最早，无法处理循环以来

Set 方法注入

属性注入





> JVM的垃圾回收算法

标记清除，标记整理，复制



> 10.jvm启动参数 你了解那些

mxs mxx，说作用感觉可以

CMS 可以配置老年代GC阈值，对象动态年龄，是否开启分配担保，是否使用显式GC，开启各个 GC

 

> 11.redis为什么快 快的原因

纯内存，优秀的数据结构，IO多路复用



> 12.redis的数据结构及实现 zset的顺序是怎么实现的

String，List，Set，ZSET，Hash

ZSET 的底层时 SkipList + ht（还有 ziplist 的实现

SkipList 的实现和 Java 基本一致。



> 13.有去了解过跳表吗

和 Java 基本一致。

实现上每次最多高一层索引，随机 long 数字，最高位和最低位为0创建索引，连续几个1创建几层最高多一层



> 14.redis集群了解

公司用 Codis，了解 redis Clsuter

都是使用的虚拟槽实现的，通过对 Key 的 Hash 确定槽位并且确定属于哪个节点。





> 15.哨兵模式为了解决什么问题

解决单点故障问题，哨兵会建立包含主从节点和其他哨兵在内的整个集群的图谱，在客观下线节点之后经过 Raft 式的选举选除 Sentinel 来进行故障转移，选择数据最全的生为主节点，原主节点退化为从节点。



> 16.mysql的mvcc了解过吗？

MVCC 中文叫做多版本并发控制，InnoDB 中用于提升并发度，只在 RR 和 RC 模式下生效。

每次的更新操作会在数据行中留下一个操作事务Id和回滚指针，查询时不会主动上锁（除非 for update，从当前数据行开始网上遍历恢复直到找到一行可以看到的数据。

表现形式就类似于快照读，但在 RR 和 RC 有不同的表现形式





> 17.mysql的锁

InnoDb 包含表锁，行锁，间歇锁，自增锁，意向锁，Next-Key锁

> 18.共享锁、排他锁、间隙锁理解

读写锁，间歇锁锁住索引间隙，next-key 是间歇锁+行锁，左闭右开区间



> 19.做更新的时候会锁住哪些数据

更新时锁住对应的主键索引，会从索引树的行记录开始锁。





> 20.怎么去优化更新的时候的锁的粒度

尽量走索引，