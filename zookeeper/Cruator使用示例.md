# Curator

---

[TOC]

---



## Introduce

Curator 是 Apache 的 Zookeeper 客户端，由 Java 语言实现，包含了各类的实用 API（好像最开始也是 Netflex 的。

maven 依赖如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.chenqwwq</groupId>
    <artifactId>zk_demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <curator.version>5.2.1</curator.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>${curator.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>${curator.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-x-discovery</artifactId>
            <version>${curator.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>${curator.version}</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.11</version>
        </dependency>

    </dependencies>

</project>
```

 

curator-framework 包含了 Curator 的核心类。

curator-recipes 则包含了大部分包装 Api，curator-x-discovery 则是服务发现相关的包（基本用不到。





## Basic

Curator 的使用基于 CuratorFramework 对象（基本就是 Client 对象。

```java
CuratorFramework zk = CuratorFrameworkFactory.builder()
  .connectString("121.44.10.23:2181") // 指定服务地址和端口
  .sessionTimeoutMs(5000)							// 会话超时时间
  .connectionTimeoutMs(5000)					// 连接超时时间
  .retryPolicy(new ExponentialBackoffRetry(1000, 3))		// 命令重试策略
  .namespace("chenqwwq")							// 命名空间，所有的命令执行都会在 /chenqwwq 
  .build();														// 创建最后的 Client 对象，也可以使用 buildTemp 创建临时对象 CuratorTempFramework
// 另外可以置顶 acl，auth，compression 规则等，调试情况还可以设定默认数据

zk.start();														// 打开客户端创建连接
```

### Create

```java
zk.create()
  .creatingParentsIfNeeded() 			// 连带创建父节点
  .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)			// 节点类型（持久/临时，普通/顺序，带TTL的节点
  .inBackground(new BackgroundCallback() {			// 是否异步执行，带回调函数
    @Override
    public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {
    }
  })
  .forPath("/qqq/locks");					// 创建的具体地址
```



## Locks

分布式锁的实现，不过这里的实现包含了更多的形式。





## Leader Elections

领导选举包含两类：

- LeaderSelector - 利用 InterProcessMutex 类来实现分布式锁，来完成选举
- LeaderLatch - 利用临时顺序节点



## Caches

尝试将远程的数据在本地缓存。



## Atomic





## Watcher





## Reference

- [https://curator.apache.org/curator-recipes/index.html](https://curator.apache.org/curator-recipes/index.html)