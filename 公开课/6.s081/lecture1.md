# 常见系统调用

int read(fd，buf，size)

> 0 表示标准的控制台输入
>
> 1 表示标准的控制台输出 

write(fd，buf，size)



int fork()

> 开启一个新的线程，新线程返回0，旧线程返回真实 pid
>
> 父子线程具有相同的堆栈环境，父线程中如果打开 fd，子线程也可以使用





exec(command,argv)

> 会丢弃当前堆栈信息，转而执行 command



命令成功执行返回0，失败则为负数



wait() 

> 等待子线程的结束，并且返回子线程的退出状态
>
> ```c
> int pid,status;
> pid = fork();
> // 等待 fork的子线程执行结束
> if(pid == 0){
>   exit(-1);
> }else{
> 	wait(&status);
>   // status = -1
> }
> exit(0);
> ```
>
> 子线程无法等待父线程结束。





## gdb 调试配置

配置 `{HOME}/.gdbinit` 增加配置项

```
add-auto-load-safe-path /Users/chenbingxin/stu/xv6-labs-2021/.gdbinit // 添加调试目录为安全目录
```

左窗口打开 `make qemu-gdb`，右窗口打开 `riscv64-unknown-elf-gdb`。

file user/_ls

break "ls.c:22"

break main

step - 下一步

c - 下个断点