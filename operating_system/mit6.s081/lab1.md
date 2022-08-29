## Lab1



Lab1 就是熟悉各类系统调用，例如 fork、exec、open、read、write 等。

主要还是对 C 语言的字符串相关语法不熟悉，所以写得有点慢。





## Sleep 

简单调用 Sleep。

1. 实现 user/sleep.c
2. 添加到 Makefile 的 UPROGS=\ 中

具体实现如下：

```c
#include "../kernel/types.h"
#include "../kernel/stat.h"
#include "../user/user.h"

#define STD_IN 0
#define STD_OUT 1
#define STD_ERR 2

int
main(int argc,char *argv[]){
  if(argc != 2){
     fprintf(STD_ERR,"Usage: sleep time\n");
     exit(1);
  }
  sleep(atoi(argv[1]));
  exit(0);
}

```





## pingpong

熟悉 fork、pipe、wait 等系统调用。

（pipe 的系统调用和 Go 的 Channel 有相似之处，emmm，可能 Go 就是基于这类实现的，算是一种同步机制。

```c
int fd[2];
pipe(fd);  // 将参数的两个文件描述符指向管道，0读1写

int pid = fork(); 
// pid == 0 的就是子线程，父线程会返回子线程的 pid

char buf[1024];
int len = read(fd[0],buf,1024); // 从 fd[0] 中读取信息，返回表示读取的字节长度

write(fd[1],buf,1024);   // 写入 buf 到 fd[1]
```

具体实现如下：

```c
#include "../kernel/types.h"
#include "../kernel/stat.h"
#include "../user/user.h"


#define STD_IN 0
#define STD_OUT 1
#define STD_ERR 2

#define PIPE_READ 0
#define PIPE_WRITE 1
int
main(){
  // parent write to p1
  // child write to p2
  int p1[2],p2[2];
  char buf[10];
  pipe(p1);
  pipe(p2);
  int pid;
  if((pid = fork()) < 0){
	  printf("fork error!\n");
	  exit(1);
  }
  if(pid == 0){
    // child process
    close(p1[PIPE_WRITE]);
    close(p2[PIPE_READ]);
    read(p1[PIPE_READ],buf,1);
    close(p1[PIPE_READ]);
    printf("%d: received ping\n",getpid());
    write(p2[PIPE_WRITE],"data",4);
    close(p2[PIPE_WRITE]);
  }else{
    // parent process
    close(p1[PIPE_READ]);
    close(p2[PIPE_WRITE]);
    write(p1[PIPE_WRITE],"data",4);
    close(p1[PIPE_WRITE]);
    read(p2[PIPE_READ],buf,1);
    printf("%d: received pong\n",getpid());
    close(p2[PIPE_READ]);
  }
  exit(0);
}

```







## primes 

使用 fork、pipe、read、write 等实现一个素数筛。

（这个素数筛的实现对我个人来说有点新颖，所以还是看了蛮久文章开始写的。

[CSP 并发模型介绍](http://swtch.com/~rsc/thread/)

具体实现还是看代码吧，如下：

```c
//
// Created by chenqwwq on 2022/7/4.
//

#include "../kernel/types.h"
#include "../kernel/stat.h"
#include "../user/user.h"

#define STD_IN 0
#define STD_OUT 1
#define STD_ERR 2

#define PIPE_READ 0
#define PIPE_WRITE 1

void primes(int fd[2]) {
    close(fd[PIPE_WRITE]);
    // 读取初始的素数
    int n;
    read(fd[PIPE_READ], &n, sizeof(n));
    printf("prime %d\n", n);

    int ffd[2];
    int m, state = 0;
    while (read(fd[PIPE_READ], &m, sizeof(m)) > 0) {
        if(m == -1) break;
        if (m % n != 0) {
            if (state == 0) {
                state = 1;
                if (pipe(ffd) < 0) {
                    fprintf(STD_ERR, "pipe error\n");
                    exit(1);
                }
                int pid = fork();
                if (pid < 0) {
                    fprintf(STD_ERR, "fork error\n");
                    exit(1);
                } else if (pid == 0) {
                    close(fd[PIPE_READ]);
                    primes(ffd);
                    exit(0);
                } else {
                    close(ffd[PIPE_READ]);
                }
            }
            write(ffd[PIPE_WRITE], &m, sizeof(m));
        }
    }
    int end = -1;
    write(ffd[PIPE_WRITE],&end, sizeof(int));
    close(ffd[PIPE_WRITE]);
    wait(0);
}

int
main() {
    int fd[2];
    if (pipe(fd) < 0) {
        fprintf(STD_ERR, "pipe error\n");
        exit(1);
    }
    int pid = fork();
    if (pid < 0) {
        fprintf(STD_ERR, "fork error\n");
        exit(1);
    } else if (pid > 0) {
        // 最开始的进程直接喂数据
        close(PIPE_READ);
        for (int i = 2; i <= 35; i++) {
            write(fd[PIPE_WRITE], &i, sizeof(i));
        }
        int end = -1;
        write(fd[PIPE_WRITE],&end,sizeof(int));
        close(fd[PIPE_WRITE]);
    } else {
        primes(fd);
        exit(0);
    }
    wait(0);
    exit(0);
}
```





## find

主要使用 open、read 等，并且熟悉相关的文件实现。

文件打开之后，会返回一个 fd，表示一个文件描述符，

```c
int fd = open(path,0); // path 表示文件路径，后面表示操作的掩码，0 表示 READ ONLY

struct stat st;
fstat(fd,&st);  // 返回1表示读取成功，stat 表示一个文件的具体信息，在内核实现 /kernel/stat.h 中
```

因为 xv6 和 Linux 一样对各类数据都是用文件的定义进行高度抽象，所以 stat 需要根据 type 来区分文件类型：

```c
// /kernel/stat.h
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device

struct stat {
  int dev;     // File system's disk device，硬盘设备
  uint ino;    // Inode number，文件号，全局唯一
  short type;  // Type of file，文件类型
  short nlink; // Number of links to file，创建的链接数量
  uint64 size; // Size of file in bytes 文件大小
};
```

这里其实又个概念，我也不确定理解的对不对，文件的在内核中的主体就是 inode，所有其他的展示都是以 link 的形式（Linux 还有软链和硬链的区别。

（这里我才意识到 debug log 的关键，c 的字符串处理真的差太多了，这里还用到了 **strcmp** 以及 `strcpy`

```c
//
// Created by chenqwwq on 2022/7/5.
//

#include "../kernel/types.h"
#include "../kernel/stat.h"
#include "../user/user.h"
#include "../kernel/fs.h"

#define O_RDONLY 0

#define debug 0

// 字符串处理才是最大的问题
// 不考虑 . 和 .. 已经各种相对路径

int cmp(char *s1, char *s2) {
    return strcmp(s1 + strlen(s1) - strlen(s2), s2);
}

void find(char *path, char *pattern) {
    if (debug) {
        printf("find:find path:[%s],pattern:[%s]\n",path,path);
    }
    int fd;
    struct stat st;
    char buf[512];
    struct dirent dir;

    if ((fd = open(path, O_RDONLY)) < 0) {
        fprintf(STD_ERR, "find: cannot open %s\n", path);
        return;
    }
    if (fstat(fd, &st) < 0) {
        fprintf(STD_ERR, "find: cannot fstat %s\n", path);
        close(fd);
        return;
    }
    switch (st.type) {
        case T_FILE:
            // 判断文件名是否相等
            if (cmp(path, pattern) == 0) {
                fprintf(STD_OUT, "%s\n", path);
            }
            break;
        case T_DIR:
            // 递归查找下一个
            // 拷贝前缀
            strcpy(buf, path);
            // 构造输出字符地址，path + '/'
            char *l = buf + strlen(path);
            *l++ = '/';
            while (read(fd, &dir, sizeof(dir)) == sizeof(dir)) {
                if (dir.inum == 0) continue;
                // l 不移动
                memmove(l, dir.name, DIRSIZ);
                l[DIRSIZ] = 0;
                if (debug) {
                    printf("find: next [%s]\n", buf);
                }
                if (cmp(l, "/.") != 0 && cmp(l, "/..") != 0) {
                    find(buf, pattern);
                }
            }
            break;
    }
    close(fd);
}

int main(int argc, char *argv[]) {
    if (debug) {
        printf("find: argv[1] %s\n", argv[1]);
        printf("find: argv[2] %s\n", argv[2]);
    }
    if (argc != 3) {
        fprintf(STD_ERR, "usage: find path pattern\n");
        exit(1);
    }
    find(argv[1], argv[2]);
    exit(0);
}
```







## xargs 

主要还是对 C 的字符串的处理，太他娘的麻烦了。

以下命令中：

```c
 echo "1\n2" | xargs -n 1 echo line
```

`|` 的实现是在 user/sh.c 中，会使用 fork 来执行前后两个命令，并且使用 pipe 来连接前后两个进程。

这里 pipe 的连接方式就很有意思了，xv6 的线程会默认开启三个线程：

1. 0 - 标准输入
2. 1 - 标准输出
3. 2 - 异常输出

并且在创建新的 fd 的时候会使用当前最小未使用的，例如你关闭了标准输入 0，在 open 一个文件，此时的文件序号就是 0。

在这里的实现就是先关闭标准输入 0 或者标准输出 1，然后 dup 出 pipe 返回的 fd。

在执行 xargs 的时候需要从标准输入在读取后续参数，此时的标准输入连接的其实是前一个命令的标准输出。

<br>

具体实现如下：

```c
//
// Created by chenqwwq on 2022/7/6.
//

#include "../kernel/types.h"
#include "../kernel/stat.h"
#include "../user/user.h"

#define STD_IN 0
#define STD_OUT 1
#define STD_ERR 2

#define DEBUG 0

void runcmd(char *cmd, char *args[], int argc) {
    if (DEBUG) {
        printf("xargs#runcmd: cmd -> [%s]\n", cmd);
        for (int i = 0; i < argc; i++) {
            printf("xargs#runcmd: agrv -> [%s]\n", args[i]);
        }
    }
    if (fork() == 0) {
        exec(cmd, args);
        exit(0);
    }
}

int
main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(STD_ERR, "usage: xargs [command]\n");
        exit(1);
    }
    // argv[0] 就是 xargs
    char buf[1024], *args[128];
    for (int i = 1; i < argc; i++) {
        if (DEBUG) {
            printf("argv%d:[%s]\n", i, argv[i]);
        }
        args[i - 1] = argv[i];
    }
    args[argc - 1] = buf;
    int idx = 0;
    while (read(STD_IN, buf + idx, 1) != 0) {
        if (buf[idx] != '\n') idx++;
        else {
            buf[idx] = 0;
            runcmd(argv[1], args, argc);
            idx = 0;
        }
    }
    if (idx != 0) {
        runcmd(argv[1], args, argc);
    }
    // 当前线程没有子线程的时候返回-1
    while (wait(0) != -1) {}
    exit(0);
}
```





## 总结

写完 Lab1，感觉最大的阻碍还是 C 语言基础。

另外需要注意的就是 argc 包含了你需要执行的命令，例如：

```
echo "hello"
```

在调用 main 方法的时候，argc 其实是2，并且 argv[0] 为 echo，argv[1] 为 hello。