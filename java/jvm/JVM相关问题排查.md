# JVM 相关问题排查





## 线程相关问题

线程相关还是使用 jstack 输出当前 JVM 下的所有线程的状态。



### 特定处理命令

先输出 jstack 的结果：**jstack [pid] > jstack.log** 

| 需求                         | 命令                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| 统计线程个数                 | grep "Thread.State" jstack.log \| wc -l                      |
| 统计各个状态的个数           | grep "Thread.State" js.log \| awk '{print$2$3$4$5}' \| sort \| uniq -c |
| 获取线程名统计个数并降序排列 | grep "daemon" 1.txt \|awk -F '[ "]' '{print $2}' \| sort ｜uniq-c \| sort -k1,1nr |





## 堆相关的文推

常用的有 jmap，jstat 等命令。



jmap -heap [pid] 查看整个堆的情形

jmap -histo [pid] 可以查看堆中存活的对象及其大小

jmap -histo:live [pid] 可以查看堆中存活的对象大小（会触发 Full GC