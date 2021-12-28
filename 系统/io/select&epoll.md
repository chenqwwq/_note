# select 和 epoll



## select

### 函数声明

函数签名如下:

```c
#include <sys/select.h>
#include <sys/time.h>

// 参数说明：maxfdp1表示监听描述符个数，监听描述符集( 读/写/异常描述符集 )，超时时间
int select ( int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout )
```

select 的函数签名中包含了监听的描述符的个数，以及需要监听的 read/write/except 三类事件的描述符，最后带上超时时间。

个人理解上，select 就是将对应的描述符交给内核监控，但是为了不被用户线程各类操作影响，所以此时就需要复制所有的描述符到内核空间。

另外需要注意的是，select 可以监听的最大的描述符被定为 1024。（poll 相比于  select 仅仅是取消了上限。

另外在超时时间内返回的时候，还需要将描述符集合移动到用户空间，并且需要手动遍历所有描述符确定具体哪个触发事件。

> select 的缺点显而易见：
>
> 1. 调用带有大量的数据复制（描述符需要在用户空间和内存空间中复制
> 2. 需要遍历全部的描述符，来确定事件（即使是 poll，对 fd_set 的遍历也是 O(n) 的时间复杂度





## epoll

```c
// 建立一个epoll对象(在epoll文件系统中给这个句柄分配资源)；
int epoll_create(int size);  
// 向 epoll 对象中添加/修改连接的套接字；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
//  收集发生事件的连接。
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```

<br>

epoll 中将原本的 select 单个的系统调用方法，拆分成了三个，并且保存有红黑树以及一个链表结构。

使用 epoll_create 方法创建 epoll 对象的同时会初始化一个红黑树结构，该结构用来保存所有需要监听的 FD（相对于完全二叉树，红黑树更适合于添加和查询。

对于任何在红黑树上的 FD，都会注册一个回调函数，在事件触发之后会调用回调返回（Callback），将 FD 添加到链表结构中。

epoll_ctl 用于添加或者修改 FD，而 epoll_wait 则是等待链表只能够是否有存在对应的监听事件发生。

<br>

相对于 select，epoll 不需要在内核空间和用户空间之间复制大量的描述符，并且使用的红黑树结构使 CRUD 的时间复杂度都得到优化。

<br>

epoll 存在边缘触发和水平触发两种模式。