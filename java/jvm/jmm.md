# Java 内存模型（JMM）



JMM（Java 内存模型）就是 JVM 为了屏蔽底层不同硬件而定义的内存使用规则。

JMM 将内存分为主内存和工作内存，主内存所有线程共享，而工作内存由线程专有。



内存访问存在以下几种性质：

1. 原子性
2. 可见性
3. 顺序性



JMM 定义了八种原子性的指令：

1. lock / unlock
2. read / load
3. store / write
4. use
5. assign



另外还有　Happen-Before 原则用于辅助判断：

1. 管程锁定原则 - lock 肯定先于 unlock 执行
2. 程序次序原则 - 单线程下确保程序执行的相对顺序
3. 线程的开启，终止，中断原则
4. 传递性原则
5. volatile 原则
6. 对象终结（finalize()）原则





## volatile 关键字

volatile 保证了变量的可见性，在 a 线程修改变量 x 之后，b 线程能立马看到。





































































