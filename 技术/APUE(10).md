---
date: 2019-07-18
status: public
tags: APUE
title: APUE(10）
---

# 1 信号概念
信号提供了处理异步事件的方法，所有信号的名称都是以 `SIG` 开头，产生信号的方法有：
1. 当用户按某些终端键；
2. 硬件异常产生信号：除数为 0， 无效的内存引用；
3. 进程调用 `kill` 函数将任意信号发送给另一个进程或者进程组；
4. 当检测到某种软件条件已经发生；

对于某种信号，可以告诉内核按照下面 3 中方式进行：
1. 忽略此信号，大部分信号可以采用这种方式，处理 `SIGKILL` 和 `SIGSTOP` 两种；
2. 捕捉信号，通知内核在某种信号发生的视乎，调用一个用户函数；
3. 执行系统默认动作；大部分信号系统默认操作是终止该进程，也有些生成 `core` 文件。

# 2 `signal` 函数
函数原型 
```c
#include <signal.h>
void (*signal (int signo, void (*func)(int)))(int);
```
`signo` 参数是信号名，`func` 的值是常量 `SIG_IGN`,`SIG_DFL` 或者是此信号调用的函数地址。
`SIG_IGN` 表明是内核忽略此信号；`SIG_DFL` 表明接到此信号后默认动作。

# 3 中断的系统调用
将系统调用分为两类：低速系统调用和其他系统调用，低速系统调用是可能会使进程永远阻塞的一类系统调用。在执行信号中断后，可以重新调用。

# 4 可重入函数
为了防止在处理信号返回时正确执行正常指令序列不一致问题，`Unix` 中包含了信号处理程序汇总保证调用安全的函数，称为**异步信号安全**，而且在信号处理期间，会阻塞任何引起不一致的信号发送。在信号处理的时候首先保存 `errno` 的值，在处理完毕后恢复 `errno` 值。

# 5 可靠信号术语和语义
向进程递送一个信号，在信号产生 (generation) 和递送 (delivery) 之间的时间间隔称为信号未决的`pending`。进程可以选择**阻塞信号递送**，如果为进程产生了阻塞的信号，而且对该信号默认动作或者捕获该信号，则信号为未决状态。每个进程都有一个信号屏蔽字，规定了当前要阻塞递送到该进程的信号集。

# 6 `kill` 和 `raise` 函数
函数原型
```c
# include <signal.h>
int kill(pid_t pid, int signo);
int raise(int signo);
```
`kill` 函数将信号发送给进程或者进程组，而 `raise` 函数将信号发送给自身。
`kill` 函数中的 `pid` 参数有一下 4 中不同的情况
- pid > 0: 将信号发送给进程 ID 为 pid 的进程；
- pid == 0: 将信号发送给同属同一进程组的所有进程；
- pid < 0: 将信号发送给进程组 ID 等于 pid 绝对值，并且具有发送信号的权限；
- pid == -1: 将信号发送给所有有权限的进程。

# 7 信号集
信号集（`signal set`) 数据类型能够表示多个信号，信号集相关的函数
```c
# include <signal.h>
int sigemptyset(signset_t *set);
int sigfillset(signset_t *set);
int sigaddset(signset_t *set, int signo);
int sigdelset(signset_t *set, int signo);
int sigismemeber(const signset_t *set, int signo);
```
`sigemptyset` 初始化一个 `set` 集合，清楚所有的信号；`sigfillset` 初始化有 `set` 执行的集合；一旦初始化完毕，可以往集合中添加，删除信号，也可以判断某个信号是否在集合中。