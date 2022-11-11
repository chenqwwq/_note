# Lab2 



Lab2 是实现一个系统调用（不可避免的知道 xv6 中系统调用的具体流程。



## XV6 系统调用流程

总体流程如下，以 trace 为例：

```
user/user.h:		用户态程序调用跳板函数 trace()
user/usys.S:		跳板函数 trace() 使用 CPU 提供的 ecall 指令，调用到内核态
kernel/syscall.c	到达内核态统一系统调用处理函数 syscall()，所有系统调用都会跳到这里来处理。
kernel/syscall.c	syscall() 根据跳板传进来的系统调用编号，查询 syscalls[] 表，找到对应的内核函数并调用。
kernel/sysproc.c	到达 sys_trace() 函数，执行具体内核操作
```



 

XV6 使用的应该是宏内核的方式，所以大部分核心功能都需要走系统调用实现，常见的包括 fork，read，write，pipe 等等。

使用系统调用就会涉及到从用户空间到内核空间到切换，又涉及到了各个堆栈信息的保存。

<br>

系统调用的方法声明在 user/user.h 中，但是具体的开端是在 usys.pl 中，该文件使用 perl 自动生成了汇编逻辑，在生成前结构如下：

```perl
print "# generated by usys.pl - do not edit\n";

print "#include \"kernel/syscall.h\"\n";

sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
## 使用 perl 生成的最终的汇编
## 生成全局的调用方法，所有系统调用需要在下面声明

entry("fork");
entry("exit");
entry("wait");
entry("pipe");
entry("read");
entry("write");
entry("close");
```

所以如果新增一个系统调用也需要在文件末尾加上 entry("xxx")，生成对应的系统调用。







## sysinfo

该系用调用主要流程和 trace 差不多，但是涉及到内核空间到用户空间的内存拷贝。