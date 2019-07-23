---
date: 2019-07-22
status: public
tags: APUE
title: APUE(12)
---

# 1 线程限制
系统对于线程的限制可以通过 `sysconf` 函数查询
- `PTHREAD_DESCRIPTOR_ITERATEIONS`: 线程退出时系统实现试图销毁线程特定数据的最大次数
- `PTHREAD_KEYS_MAX`：进程可以创建键最大的数目
- `PTHREAD_STACK_MIN`：一个线程栈可用的最小字节
- `PTHREAD_THREADS_MAX`：进程可以创建的最大线程数

# 2 线程属性
- 每个对象与它自己类型的属性对象关联，每个属性对象包含多个属性，而且对象对于应用程序是透明的，只提供操作的函数。
- 一个初始化函数，设置为默认值；
- 一个销毁属性对象的函数，销毁分配的资源；
- 每个属性都有从属性对象中获取值的函数；
- 每个属性都有一个设置属性值的函数；

# 3 同步属性
## 3.1 互斥量属性
属性用 `pthread_mutexattr_t` 结构表示
```c
# include <pthread.h>
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
int pthread_mutexattr_destrory(pthread_mutexattr_t *attr);
```
值的注意的是 3 个属性：
1. 进程共享属性，其值为 `PTHREAD_PROCESS_PRIVATE`, 允许线程库提供更有效的互斥量实现，
2. 健壮性属性，当互斥量进程终止的时候，需要解决互斥量状态恢复的问题。
3. 类型属性，控制互斥量锁定的特性

## 3.2 读写锁属性
使用 `pthread_rwlockattr_t` 类型表示，唯一支持的属性是**进程共享**属性。

## 3.3 条件变量属性
使用 `pthread_condattr_t` 类型表示，支持**进程共享**属性，控制条件变量可以被单进程多个线程使用，还是可以被多进程的线程使用。
支持**时钟**属性，用来指定超时使用的哪个时钟。

## 3.4 屏障属性
使用 `pthread_barrierattr_t` 类型表示，只支持**进程共享**属性。

# 4 重入
如果一个函数在同一时间点可以被多个线程安全使用，那么该函数是线程安全的。对于一些非线程安全的函数，会提供可替代的线程安全的版本。如果一个函数对于多个线程是可重入的，就说这个函数是线程安全的，但是这并不能说明对信号处理程序来说该函数是可重入的。

# 5 线程特定数据
线程特定数据（`thread-specific data`)，也称为线程私有数据（`thread-private data`)是存储和查询某个特定线程数据的一种机制。在分配线程特定数据之前，需要创建于该数据关联的键。这个键用于获取对该线程特定数据的访问。
```c
#include<pthread.h>
int pthread_key_create(pthread_key_t *keyp, void (*destructor)(void *));
```
创建的键存储在 `keyp`指向的内存单元，这个键可以被进程中所有的线程使用，但是每个线程把这个键与不同的线程特定数据地址进行关联，`destructor` 指定这个键关联的析构函数。

# 6 取消选项
可取消状态属性可以是 `PTHREAD_CANCEL_ENABLE` 或者 `PTHREAD_CANCEL_DISABLE`
```c
# include <pthread.h>
int pthread_setcancelstate(int state, int *oldstate);
```
`pthread_cancel` 并不等待线程终止，默认线程在取消请求发出后还是继续运行，知道线程到达某个取消点。通常在调用某些函数的时候会测试该线程是否到达取消点。

# 7 线程和信号
每个线程都有自己的信号屏蔽字，但是信号的处理是进程中所有的线程共享的。这意味着单个线程可以阻止某些信号，但是当其他线程修改后，所有的线程必须共享这个处理行为的改变。
进程中的信号是投递到单个线程中，如果一个信号与硬件相关，那么该信号发送至引起该事件的线程中去，其他信号则发送至任意一个线程。可以通过 `pthread_kill` 将信号发送给线程
```c
#include <signal.h>
int pthread_kill(pthread_t thread, int signo);
```

# 8 进程和 `fork`
在线程中调用 `fork`，就为子进程创建了整个进程的地址空间的副本，而且从父进程那继承了每个互斥量、读写锁和条件变量。在子进程内部，只有一个线程，就是父进程中调用 `fork` 的线程副本构成。如果父进程中的线程占有锁，子进程也将占有同样的锁，问题是子进程并不包含占有锁的线程副本，所以子进程并不知道它占有了哪些锁，需要释放哪些锁。
为了避免不一致情况，子进程只能调用异步信号安全的函数，要清除锁状态，调用 `pthread_atfork` 函数建立 `fork` 处理程序。
```c
#include <pthread.h>
int pthread_atfork(void (*preapre) (void), void (*parent) (void), void (*child) (void));
```
- `prepare` fork 处理程序有父进程在 `fork` 创建子进程前调用，主要是获取父进程定义的所有锁；
- `parent` fork 创建子进程以后、返回之前在父进程上下文使用，对 `prepare` fork 处理程序获取的所有锁进行解锁。
- `child` fork 处理程序在 `fork` 返回之前在子进程上下文中调用。用来释放 `prepare` 处理程序获取的锁。

# 9 线程和 `I/O`
`pread` 和 `pwrite` 是线程安全的 `I/O` 读写。