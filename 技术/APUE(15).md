---
date: 2019-07-31
status: draft
tags: APUE
title: APUE(15）
---

# 1 管道
管道具有以下局限性
1. 历史上，管道都是半双工的（数据只能在一个方向上流动）
2. 管道只能在具有公共祖先的的进程之间使用。

创建管道
```c
# include <unistd.h>
int pipe(int fd[2]);
```
`fd[1]` 的输出是 `fd[0]` 的输入。单进程的管道没有意义，通常进程首先调用 `pipe`，接着调用 `fork`，从而创建父进程到子进程的 `IPC` 通道。然后关闭各自相关的描述符构建数据流方向。
![](./_image/2019-07-31-19-42-35.jpg?r=62)
当管道一端被关闭
1. 当读一个写端已经被关闭的管道，在所有的数据被读取后，read  返回 0
2. 当写一个读端硬被关闭的通道，则产生信号 `SIGPIPE`

# 2 `popen` 和 `pclose` 函数
函数原型
```c
#include <stdio.h>
FILE *popen(const char *cmdstring, const char *type);
int pclose(FILE *fp);
```
函数 `popen` 先执行 `fork`, 然后调用 `exec` 执行 `cmdstring`，并且返回一个标准的 `I/O` 文件指针，如果 `tyep` 是 `r`，则文件指针连接到 `cmdstring` 的标准输出；如果 `type` 是 `w`，则文件指针链接到 `cmdstring` 的标准输入。

# 3 协同进程
当一个过滤程序既产生某个过滤程序的输入，又读取过滤程序的输出时，就是协同进程。

# 4 FIFO
`FIFO` 又称命名管道，是一种文件类型，不相关的进程也能交换数据。
```c
# include <sys/stat.h>
int mkfifo(const char *path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
```

非阻塞标志
- 在一般情况下(没有指定 `O_NONBLOCK`)，只读的 open 要阻塞其他继承为写而打开；
- 如果指定了 `O_NONBLOCK`，只读的 open 立即返回，如果没有进程没有为其打开，则 `open` 返回 -1.

# 5 XSI IPC
主要包含消息队列、信号量和共享存储器。

## 5.1 标识符和键
内核中的每个 IPC  都使用非负整数加以标识，每个IPC 对象都有一个键（key）关联，这个键作为对象的外部名称。
客户进程和服务端进程在同一个 IPC 结构上汇聚
- 服务进程将返回的标识符存放在某个地方（文件）以便客户进程使用；
- 在共用的头文件定义一个服务进程和客户进程都认可的键
- 客户进程和服务进程认同一个路径名和项目 ID。

## 5.2 缺点
- IPC  没有引用计数；
- IPC 结构在文件系统的灭于名字，常见的文件操作并没有作用，需要特别命令处理；
- 不能使用文件描述符，不能使用多路复用的 `I/O` 函数。

# 6 消息队列
- msgget 用于创建新的队列或者打开现有的队列；
- msgsnd 用于将消息发送至队列；
- msgrcv 用于从队列中获取消息；

每个消息有三个部分组成：一个正的长整类型的字段、一个非负的长度，以及实际的字节数。

# 8 信号量
它是一个计数器，用于多个进程提供共享对象的访问。
需要完成的操作
1. 测试控制该资源你的信号量
2. 若次信号量为正，则可以使用资源，信号量减1
3. 如果此信号量为 0， 则进入休眠状态；直至信号量大于 0， 唤醒该进程；
4. 如果进程不再需要改资源，则信号量加 1 ；

# 9 共享存储
多个进程共享给定的存储区，不需要数据复制，因此最快。
```c
# include <sys/shm.h>
int shmget(key_t key, size_t size, int flag);
```
进程可以连接到它的地址空间
```c
# include <sys/shm.h>
void *shmat(int shmid, const void *addr, int flag);
```

