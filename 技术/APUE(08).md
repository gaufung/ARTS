---
date: 2019-07-13
status: draft
tags: APUE
title: APUE(08)
---

# 1 进程标识
每个进程都有一个非负整数标识唯一进程 ID，`Unix` 进程的 ID 是复用的，但是是延迟复用算法。
- ID 为 0 的进程是调度进程，是内核的一部分，不执行任何磁盘上的程序。
- ID 为 1 的为 `init` 进程，通常读取与系统有关的初始化文件，该进程不会终止，是普通的用户进程，也是所有孤儿进程的父进程。
- ID 为 2 的是页守护进程，支持虚拟存储器系统分页操作。

# 2 `fork` 函数
现有进程调用 `fork` 函数可以创建新的进程
```c
# include <unistd.h>
pid_t fork(void);
```
创建的新的进程称为子进程，`fork` 函数被调用一次， 但是返回两次。子进程返回值为 `0`, 而父进程返回子进程的 ID。父子进程返回的顺序不确定，取决于调度算法。
`fork` 的时候父进程所有打开的文件描述符都复制到子进程中，除此之外还有子进程继承过来的。
- 实际用户ID，实际组ID，有效用户ID，有效组ID
- 附属组ID
- 进程组 ID
- 会话ID
- 控制终端
- 设置用户 ID 标志和设置组 ID 标志
- 当前工作目录
- 根目录
- ...

**区别**
- `fork` 返回值不同
- 进程 ID 不同
- 进程父进程 ID 不同
- 子进程的 `tms_utime`, `tms_stime`, `tms_cutime` 和 `tms_ustime` 设置为 0
- 子进程不继承父进程的文件锁
- ...

`fork` 的用法：
1. 父进程复制自己，分别执行不同的代码段。在网络服务中，父进程等待客户端请求，子进程处理此请求。
2. 一个进程执行不同的程序，`fork` 子进程返回后调用 `exec`。有些操作系统合并这两个步骤为 `spawn`。

# 3 `vfork` 函数
`vfork` 创建子进程，不过在父进程的空间中运行，用以提高效率；还有 `vfork` 保证子进程先运行，在调用 `exec` 或者 `exit` 之后父进程才会被调用。

# 4 `exit` 函数
**正常终止**

1. `main` 函数执行 `return` 语句
2. 调用 `exit` 函数，调用注册函数，关闭标准 `I/O` 流
3. 调用`_exit` 和 `_Exit` 函数
4. 进程的最后一个线程在其启动的时候执行 `return` 语句
5. 进程的最后一个线程调用 `pthread_exit` 函数

**异常终止**

1. 调用 `abort`
2. 进程接受某些信号
3. 最后一个线程对**取消**请求做出相应

对于父进程已经终止的所有进程，它们的父进程将会变成 `init` 进程。内核为每个终止的子进程保存了一定量的信息，所以父进程调用 `wait` 或者 `waitpid` 获取这些信息。一个已经终止，但是父进程尚未对其善后处理的进程称为僵死进程。

# 6 `wait` 和 `waitpid` 函数
当进程正常或者异常终止的时候，内核向其父进程发送 `SIGCHLD` 信号。`wait` 和 `waitpid` 则主动查看子进程状况。
1. 如果其所有的子进程都还在运行，则阻塞
2. 如果一个子进程已经终止，正在等待的父进程获取其终止状态，则取得该子进程的终止状态立即返回。
3. 如果它没有任何子进程，则返回出错。

```c
# include <sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```
区别：
- 在子进程终止前，`wait` 使其调用者阻塞；`waitpid` 有一选项，可使调用者不阻塞。
- `waitpid` 并不等待其调用之后的第一个终止的子进程。


如果一个进程有几个子进程，只要有一个进程终止，`wait` 就返回。`waitpid` 指定一个子进程返回。
- $pid==-1$： 等待任一子进程返回；
- $pid>0$: 等待子进程和 `pid` 相同的子进程；
- $pid==0$: 等待组 ID 等于调用进程组 ID 的任一子进程
- $pid<-1$：等待组ID 等于 `pid` 绝对值的任一子进程

# 7 `waitid` 函数
```c
# include <sys/wait.h>
int waitid(idtype_t idtype, id_t id, signinfo_t *infop, int option);
```

其中  `idtype` 的类型有：
- `P_PID`: 等待特定的进程
- `P_PGID`: 等待特定进程组中的任一子进程
- `P_ALL`: 任一子进程，忽略 `id`

# 8 `wait3` 和 `wait4` 函数
```c
# include <sys/types.h>
# include <sys/wait.h>
# include <sys/time.h>
# incldue <sys/resource.h>
pid_t wait3(int *statloc, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *statloc, int options, struct rusage *rusage);
```
增加了参数，用以返回子进程使用的资源情况。

# 9 `exec` 函数
进程调用 `exec` 函数时，该进程执行的程序完全替换新的程序，新程序从 `main` 函数开始执行，因为不创建新的进程，所以进程 ID 没有改变， `exec` 只是用磁盘上的程序替换当前程序的正文段、数据段、堆栈和栈。
总共有 7 种 `exec` 函数可以使用 
```c
# include <unistd.h>
int execl(const char *pathname, const char *arg0, ...,  /* (char *)0 */);
int execv(const char *pathname, char const argv[]);
int execle(const char *pathname, const char *args0, ..., /* (char *)0 char *const evp[]  */ );
int execve(const char *pathname, char *const argv[], char *const envp[]);
int execvp(const cahr *pathname, char *const argv[]);
int fexecve(int fd, char *const argv[], char *const *envo[]);
```
这些函数主要有三个区别
1. 是否使用路径名作为参数还是，绝对路径还是相对路径或者是文件描述符；
2. 传递参数形式，依次传递参数还是传递参数数组；
3. 传递环境表

![](./_image/2019-07-14-14-45-11.jpg?r=65)

7  个函数㕜，只有个 `execve`  是系统调用，其余的都是库函数。

# 10 更改用 ID 和更改组 ID 

```c
# include <unistd.h>
int setuid(uid_t uid);
int setgid(uid_t gid);
```
更改规则
1. 如果进程有超级用户的权限，则 `setuid` 函数将实际用户 ID 、有效用户 ID  以及保存的设置用户 ID 设置为 `uid`;
2. 如果进程没有超级用户特权，但是 `uid` 等于实际用户 ID 或者保存的设置用户 ID， 则 `setuid` 的有效用户 ID 设置为 `uid`，不更改实际用户 ID 和保存用户设置 ID。 
3. 都不满足，否则 `errno` 设置为 `EPERM`，并返回 `-1`。

内核维护的 3 个用户 ID ，需要的注意点：
1. 只有超级用户才能更改实际用户 ID，实际用户 ID 在登录的时候指定，而且绝对不会改变它；
2. 仅当对程序文件设置用户 ID 位的时候， `exec` 函数才设置有效用户 ID，否则 `exec` 函数不会改变有效用户 ID。
3. 保存的设置用户 ID 是由 `exec` 复制有效用户 ID 而得到的。

# 11 解释器文件
```shell
#! pathname [optional-argument]
```
`pathname` 通常使用绝对路径名，内核调用执行的进程是 `pathname` 指定的文件。

# 12 `system` 函数
```c
# include <stdlib.h>
int system(const char *cmdstring);
```
这个对系统依赖程度较大，在 `system` 实现过程中调用了 `fork`, `exec` 和 `waitpid` ，因此有三种返回值
1. `fork` 失败或者  `waitpid` 失败除了返回 `EINTR` 之外的错误，则返回 `-1`, 并且设置 `errno` 指示错误类型；
2. `exec` 失败，其返回值如同执行 `shell` 执行失败；
3. 上述函数都执行陈宫，则返回 `shell` 的终止状态。

# 13 进程会计
典型的会计记录包含总量最小的二进制数据：包含命令名，使用的 `CPU` 时间总量、用户 ID 和组 ID，启动时间等

# 14 进程调度
进程可以通过调整 `nice` 值选择更低优先级运行。
```c
# include <unistd.h>
int nice(int incr);
```
也可以使用 `getpriority` 函数获取当前的 `nice` 值
```c
# include <sys/resource.h>
int getpriority(int which, id_t who);
```

# 15 进程时间
```c
# include <sys/times.h>
clock_t times(struct tms *buf);
struct tms {
    clock_t tms_utime; // user CPU time
    clock_t tms_stime; // system CPU time
    clock_t tms_cutime; // user CPU time, terminated children
    clock_t tms_cstime; // system CPU time, terminated children
};
```