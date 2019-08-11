---
date: 2019-07-21
status: draft
tags: APUE
title: APUE(11)
---
# 1 线程概念
典型的 `UNIX` 进程只包含一个控制线程，有了多个控制线程后，有一下好处：
- 为每种事件类型单独分配处理线程，简化异步处理事件代码；
- 多个线程可以访问相同的存储地址空间和文件描述符；
- 提高整个程序的吞吐量；
- 交互式应用程序使用多线程改善应用程序相应时间；
每个线程包含所执行环境所必需的信息，包括进程中的线程 ID， 一组寄存器值，栈，调度优先级和策略， 信号屏蔽字，`errno` 变量以及栈私有数据。

# 2 线程标识
线程 ID 用 `pthread_t` 数据类型来表示，而且只有在所属的进程上下文才有意义。

# 3 创建线程
函数原型
```c
#include<pthread.h>
int pthread_create(pthread *restrict tidp, const pthread_attr_t *restrict attr, void *(*start_rtn)(void *), void *restrict arg);
```
如果 `pthread_create` 成功返回，那么创建的线程 ID 会包含在 `tidp` 所指向的内存单元，`attr` 用来指向不同的线程属性。新创建的线程从 `start_rtn` 函数地址开始执行，该函数只有一个无关类型指针的参数 `arg`, 如果需要的参数不止一个，则将这些参数放入到一个结构中。

# 4 线程终止
3 中线程终止的情况：
1. 线程简单的从启动例程中返回；
2. 线程可以被同一进城中的其他线程取消；
3. 线程调用 `pthread_exit` 函数。

```c
# include <pthread.h>
int pthread_join(pthread_t thread, void **rval_ptr);
```
调用线程一直阻塞，直到指定线程终止。如果线程是从启动例程返回，则 `rval_ptr` 则包含返回码，如果线程被取消，则 `rval_ptr` 指定的内存单位为 `PTHREAD_CANCELED`。
线程取消函数 `pthread_cancel` 
```c
# include <pthread.h>
int pthread_cancel(pthread_t tid);
```
线程可以选择忽略这个线程取消的消息，线程也可以注册清理程序。
```c
# include <pthread.h>
void pthread_cleanup_push(void (*rtn)(void *),  void *args);
void pthread_cleanup_pop(int execute);
```
清理函数 `rtn` 在下面情况下执行
- 调用 `pthread_exit` 时候；
- 响应取消请求时；
- 用非零 `execute` 参数调用 `pthreacd_clean_pop` 时。

进程原语 | 线程原语 | 描述
:---|:---|:---
`fork` | `pthread_create` | 创建新的控制流
`exit` | `pthread_exit` | 从现有的控制流退出
`waitpid` | `pthread_join` | 从控制流中得到退出的状态
`atexit` | `pthread_cancel_push` | 注册在退出控制流时调用的函数
`getpid` | `pthread_self` | 获取控制流的 ID 
`abort` | `pthread_cancel` | 请求控制流非正常退出

# 6 线程同步
当多个线程对同一个变量进行读写的时候，会产生预期不一致的现象，此时需要对线程进行同步。
## 6.1 互斥锁
同一时间内只有一个线程访问数据
```c
# include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutextatt_t attr);
int pthread_mutex_destrory(pthread_mutex_t *mutex);
```
使用互斥锁
```c
# include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlokc(pthread_mutex_t *mutex);
```
## 6.2 读写锁
读写锁非常适合对数据结构读的次数大于写的情况，在读写锁在写的情况下，能够安全得被修改。
```c
# include <pthread.h>
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

## 6.3 条件变量
条件变量允许多个线程在同一个时刻汇合，允许线程以无竞争的方式等待特定的条件发生。
等待条件发生:
```c
# include <pthread.h>
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
```
当条件发生的时候：
```c
# include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond); // awake up at least one thread.
int pthread_cond_broadcast(pthread_cond_t *cond); // awake up all threads
```
## 6.4 自旋锁
和互斥锁类似，不过不是通过休眠的形式，而是出于忙等，用于锁被持有时间较短，而且线程不希望在重新调度上花费太多时间。
有些互斥锁在实现的时候先自旋一段时间，只有自选计数达到一定数量，才进入休眠。
```c
# include <pthread.h>
int pthread_spin_lock(pthread_spinlock_t *lock);
int pthread_spin_trylock(pthread_spinlock_t *lock);
int pthread_spint_unlock(pthread_spinlock_t *lock);
```
## 6.5 屏障
用户协调多个线程并行工作的同步的机制，允许每个线程等待，知道所有合作线程到达某个临界点。
```c
#include <pthread.h>
int pthread_barrier_init(pthread_barrier_t *restrict barrier, const pthread_barrierattr_t *restrict attr, unsigned int count);
int pthread_barrier_wait(pthread_barrier_t *barrier);
```
初始化的时候 `count` 指定需要等待的线程数量。
