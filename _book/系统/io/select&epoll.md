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









## epoll

```c
// 建立一个epoll对象(在epoll文件系统中给这个句柄分配资源)；
int epoll_create(int size);  
// 向 epoll 对象中添加 连接的套接字；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
//  收集发生事件的连接。
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```

> 简单的流程叙述：

1. 使用 epoll_create 创建一个 epoll 的结构
2. 使用 epoll_ctl 将一个 fd 添加到需要监听的集合里，epoll 使用红黑树作为集合，添加的同时注册中断回调函数，在中断到来时将 fd 移动到就绪列表
3. epoll_wait 就是扫描 epoll