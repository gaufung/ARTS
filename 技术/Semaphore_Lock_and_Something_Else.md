---
title:  信号量，锁和 golang 相关源码分析
date: 2018-08-12
status: public
---
# 1资源同步
并发已经成为现代程序设计中的重要考虑内容，但是并发涉及到一个很重要的内容就是资源同步。当两个或者多个线程访问同样的资源的时候，运行的结果取决于线程运行时精确的时序。这样导致结果与期望的结果大相径庭，因此我们需要对资源的访问顺序进行控制，已达到资源同步的目的。

## 1.1 解决方案
将共享资源（也就是共享内存）的程序片段成为`临界区域`，通过适当安排，使得两个者线程同时位于临界区域。对于`临界区域`访问解决方案，需要满足如下4个条件
- 任何两个线程不能同时位于临界区
- 不对CPU执行速度和时间做任何假设
- 临界区外运行的线程不阻塞其他线程
- 不能使线程无限期等待进入临界区

常见的互斥解决方案：

**屏蔽中断**
线程的切换是由CPU中断机制提供的，如果一个线程进入临界区域后，CPU关闭中断响应；在离开临界区域后，再打开中断机制。那么在临界区域将不会有其他线程来竞争资源。 当时将屏蔽中断权利交给用户空间执行是不明智的，而且对于多核CPU而言没有效果。

**锁变量**
几乎每一个编程语言都提供了资源同步方式：锁机制。该机制通过对资源进行`Lock`和`Unlock`，以达到对关键资源有序访问。

**严格轮换法**
线程不停的执行CPU时间，连续测试某一个值是否出现。但是如果认为等待的时间非常短，可以使用该方式浪费CPU时间，用于等待的锁也成为`自旋锁`。

# 2 信号量

## 2.1 共享变量
在理解信号量之前，先了解采用共享变量使用多线程会出现什么问题。
下面是一个C代码片段
```C
for (i=0; i<niters; i++){
    cnt ++;
}
```
`cnt`为全局变量，一个线程执行该代码片段的时候的汇编代码如下：
```asm:n
    movq (%rdi), %rcx
    testq %rcx, %rcx
    jle .L2
    movl $0, %eax
.L3:
    movq cnt(%rip), %rdx
    addq %eax
    movq %eax, cnt(%rip)
    addq $1, %rax
    cmpq %rcx, %rax
    jne .L3
.L2
```
其中`6-8`行分别对应对应着加载`cnt`，更新`cnt`和存储`cnt`。将`cnt`变量从内存位置读出，加载到CPU寄存器中，在CPU运算器中加1，然后存储到`cnt`的内存位置。虽然代码中`cnt++`只有一行，但是转换为汇编代码的时候不只有一个操作，也就是说该语句不是原子操作。如果多个线程同时执行代码，按照之前的条件，不对CPU的执行顺序做任何假设，如果其中线程`a`在执行`7`行汇编代码，而线程`b`执行`6`行汇编代码，那么`b`将"看不到"线程`a`对全局变量`cnt`加1的操作，那么每次执行的结果`cnt`也不完全一致。

## 2.2 信号量
计算机领域先驱`Dijkstra`提出经典的解决上述问题的方法：信号量(semaphore)。它是一个非负整数的全局变量。而且该变量只能有两个特殊操作来处理: `P`和`V`。
- P(s)： 如果`s`非零，那么`P`将`s`减`1`，并且立即返回。如果`s`为零，那么就挂起这个线程，知道`s`为非零。
- V(s):  `V`操作将`s`加`1`。如果有任何线程阻塞在`P`操作等待`s`非零，那么`V`将重启其中线程中的一个。

`Posix`标准定义需要操作信号量的函数
```C
#include <semaphore.h>
int sem_init(sem_t *sem, 0, unsigned int value);
int sem_wait(sem_t *s); /*P(s)*/
int sem_post(sem_t *s); /*P(s)*/
```
那么如何使用信号量是的2.1小节出现同步问题解决呢？
首先定义全局信号量
```C
volatile long cnt = 0; /* global variable */
sem_t mutex; /*global semaphore*/
```
初始化信号量，在这里初始值为`1`
```C
sem_init(&mutex, 0, 1);
```
最后使用信号量操作函数将临界区域代码包含起来
```C
for (i =0; i<niters; i++){
    sem_wait(&mutex);
    cnt++;
    sem_post(&mutex);
}
```
# 3 锁
## 3.1 死锁
首先看一下死锁的规范定义:
> 如果一个线程（进程）集合中的每一个线程（进程）都在等待只能由该线程（进程）集合中的其他线程（进程）才能引发的事件，那么该线程（进程）集合是死锁的。

举一个例子，如果线程 `a` 和线程 `b` 同是执行，线程`a`获取了资源`r1`，等待获取资源`r2`；而线程`b`获取了资源`r2`，等待获取资源`r1`。那么线程`a`和线程`b`组成的集合是死锁的。

**预防死锁**

- 破坏占有等待条件
  
  对于需要获取多个资源的线程，一次性获取全部资源，而不是依次获取各个资源。

- 破坏环路等待条件
  
  死锁集合的线程按照等占有线程和等待线程可以组成有向环图。那么如果对所有资源进行排序，所有线程按照资源顺序获取资源。

## 3.2 活锁

在某些情况下，当线程意识它不能获下一个资源的时候，它会“礼貌性”地释放已经获得的资源，然后等待`1ms`，在尝试一次。如果另一个线程也在相同的时候做了相同的操作， 那么同步的步调将导致两个线程都无法前进。

## 3.3 饥饿

在信号量小节中，当执行`V`操作后，将恢复挂起线程中的一个，那么问题出现了：如果有多个线程被挂起，那么选择哪个线程恢复呢？如果随机选择一个线程恢复，如果源源不断的线程到达临界区域并且挂起，那么很有可能出现某一个线程一直等待资源，而导致"饥饿"。当然也有好的`FILO`调度策略来解决调用问题。当时问题在于刚刚到达的线程有很好的局部性，也就是CPU的寄存器、缓存等包含了该线程的局部变量，如果程获得资源锁，很好的避免了线程上下文切换，对性能提高很有帮助。

在`go`语言的互斥锁中采用结合上述两种策略，接下来小节中，将会仔细分析源码。

# 4 Golang sync 包
## 4.1 sync.mutex.go

### 4.1.1 接口和结构
包含了`Locker`接口和`Mutex`结构：
```go
type Locker interface {
	Lock()
	Unlock()
}
type Mutex struct {
	state int32
	sema  uint32
}
```
`Mutex`实现了`Locker`接口，该结构包含了`state`的字段，用来表示该锁当前状态；`sema`则为一个信号量。`state`是一个32位的整数，不同比特位包含了不同的意义，其中源码中的有很详细的注释，该注释很好解释`mutex`如何工作：
>互斥锁有两种状态：正常状态和饥饿状态。在正常状态下，所有挂起等待的goroutine按照`FIFO`顺序等待。唤醒的goroutine将会和刚刚到达的goroutine竞争互斥锁的拥有权，因为刚刚到达的goroutine具有优势：它刚刚正在CPU上执行，所以刚刚唤醒的goroutine有很大可能在锁竞争中失败。如果一个等待的goroutine超过1ms没有获取互斥锁，那么它将会把互斥锁转变为饥饿模式。在饥饿模式下，互斥锁的所有权将移交给等待队列中的第一个。新来的goroutine将不会尝试去获得互斥锁，也不会去尝试自旋操作，而是放在队列的最后一个。如果一个等待的goroutine获取的互斥锁，如何它满足一下其中的任何一个条件：(1)它是队列中的最后一个；(2)它等待的时候小于1ms。它会将互斥锁的转台转换为正常状态。正常状态有很好的性能表现，饥饿模式也是非常重要的的，因为它能阻止尾部延迟的现象。
```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota
	starvationThresholdNs = 1e6
)
```
- `mutexLocked`
该值为`1`, 第一位比特位`1`，代表了该是否该互斥锁已经被锁住。`mutex.state`与它进行`&`操作，如果为`1`表示已经锁住，`0`则表示未被锁住。
- `mutexWoken`
该值为`2`，第二位比特位`1`，代表了该互斥锁是否被唤醒，`mutex.state`与它进行`&`操作，如果为`1`表示已经被唤醒，`0`代表未被唤醒
- `mutexStarving`
该值为`4`,第三位比特为`1`，代表了该互斥锁是否处于饥饿状态，`mutex.state`与它进行`&`操作，如果为`1`表示处于饥饿转态，`0`表示处于正常状态。
- `mutexWaiterShift`
该值为`3`，表示`mutex.state`右移3位后为等待的`goroutine`的数量。
- `starvationThresholdNs`
`goroutine`将互斥锁转换状态的时间等待的临界值：一百万纳秒，也就是1ms。

### 4.1.2 Lock 方法
```go
if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
}
```
`CompareAndSwapInt32`是一个原子操作，它判断是一个参数的值是否等于第二个参数，如果相等，则将第一个参数设置为第三个参数，并返回`true`；否则对一个参数不做任何操作并且返回`false`。这一段是代码是处理第一次`goroutine`进行尝试`Lock`操作，如果一切都是初始状态，则`m.state`为`.....0000001`并且返回，进入临界区域代码，否则代码继续往下走。
```go
var waitStartTime int64
starving := false
awoke := false
iter := 0
old := m.state
```
首先定义了一下变量：`goroutine`等待时间，是否饥饿转台，是否唤醒和自旋迭代次数和保存当前互斥锁状态。
接下来是一个`for`循环，只有退出循环才能进入临界区域代码，纵观代码只有两处使用`break`来退出循环。
```go
for {
    if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
}
```
首先判断锁状态是被锁而且不是处于饥饿模式，加上还能自旋额次数，进入下一层判断。如果`当前goroutine没有被唤醒``，`其他goroutine也没有被唤醒`，`等待的goroutine超过1`和`可以将m.state设置为唤醒转态`四个条件同时满足，将`awoke`设置`true`。然后进行自旋操作，进行一轮循环。

```go
new := old
if old&mutexStarving == 0 {
			new |= mutexLocked
}
if old&(mutexLocked|mutexStarving) != 0 {
		new += 1 << mutexWaiterShift
}
if starving && old&mutexLocked != 0 {
			new |= mutexStarving
}
```
这三个判断条件做了如下工作：如果当前的`mutex.state`处于正常模式，则将`new`的锁位设置为1，如果当前锁锁状态为锁定状态或者处于饥饿模式，则将等待的线程数量+1。如果`starving`变量为`true`并且处于锁定状态，则`new`的饥饿状态位打开。

```go
if awoke {
	if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
	}
	new &^= mutexWoken
}
```
如果 `goroutine`已经被唤醒，则清空`new`的唤醒位。

```go
if atomic.CompareAndSawpInt32(&m.state, old, new){
    //...
}else{
    //...
}
```
如果更新`m.state`成功
```go
if old&(mutexLocked|mutexStarving) == 0 {
				break 
}
```
如果未被锁定并且并不是出于饥饿状态，到达第一个`break`，进入代码临界区域。
```go
queueLifo := waitStartTime != 0
if waitStartTime == 0 {
	waitStartTime = runtime_nanotime()
}
runtime_SemacquireMutex(&m.sema, queueLifo)
```
`runtime_SemacquireMutex(s *uint32, lifo bool)`函数类似`P`操作，如果`lifo`为`true`则将等待`goroutine`插入到队列的前面。在这里，对于每一个到达的`goroutine`，如果`CompareAndSawpInt32`成功，并且到达时候如果锁出于锁定状态，那么将该`goroutine`插入到等待队列的最后，否则插入到最前面。此时`goroutine`将会被挂起，等待`Unlock`的`V`操作，将唤醒`goroutines`

```go
starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
old = m.state
if old&mutexStarving != 0 {
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
}
```
判断被唤醒的线程是否为达到饥饿状态，也就是等待时间超过`1ms`，如果之前的`m.state`不是饥饿状态，继续循环，给新到来`goroutine`让出互斥锁。如果已经饥饿状态，则修改等待`goroutine`数量和饥饿状态位，并返回进入临界代码区域。

### 4.1.3 Unlock方法
```go
new := atomic.AddInt32(&m.state, -mutexLocked)
```
首先创建变量`new`，该变量的锁位为`0`。接下来是饥饿状态判断
```go
if new&mutexStarving == 0 {
		old := new
		for {
				if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false)
				return
			}
			old = m.state
		}
	} else {
		runtime_Semrelease(&m.sema, true)
	}
```
如果是正常状态，则判断如果等待的`goroutine`为零，或者已经被锁定、唤醒、或者已经变成饥饿状态，返回，不需要唤醒任何其他被挂起的`goroutine`，因为互斥锁已经被其他`goroutine`抢占。否则更新`new`值（修改等待的goroutine数量）并设置唤醒为，如果`CompareAndSwapInt32`成功，则通过`runtime_Semrelease(&m.sema, false)`恢复挂起的goroutine.r如果为 `true`表明将唤醒第一个阻塞的`goroutine`，这第一点在`else`饥饿的分支中体现。

## 4.2 sync.rwmutex.go
读写锁也是一种常见的锁机制，它允许多个线程读共享资源，只有一个线程写共享资源，接下来看看go中如何实现读写锁。

### 4.2.1 常量和结构
```go
type RWMutex struct {
	w           Mutex  
	writerSem   uint32 
	readerSem   uint32 
	readerCount int32  
	readerWait  int32 
}
const rwmutexMaxReaders = 1 << 30
```
`RWMutex`结构包含了如下的字段
- w：互斥锁，保证只有一个写`goroutine`进入临界代码
- writerSem：写信号量
- readerSem：读信号量
- readerCount: 准备读的`goroutine`的数量
- readerWait:  离开读的`goroutine`的数量

而`rwmutexMaxReaders`表明最多的读`goroutine`数量。

### 4.2.1 RLock和RUnlock方法
```go
func (rw *RWMutex) RLock() {
	// [...]
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_Semacquire(&rw.readerSem)
	}
//[...]
}
```
首先是`readerCount`值+1, 如果小于零，则挂起`goroutine`等待`readerSem`。是不是很奇怪，为什么会小于零判断呢？在这里先卖一个关子，接下来会看到为什么是这样的设计逻辑。
```go
func (rw *RWMutex) RUnlock() {
	//[...]
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		if r+1 == 0 || r+1 == -rwmutexMaxReaders {
			race.Enable()
			throw("sync: RUnlock of unlocked RWMutex")
		}
		if atomic.AddInt32(&rw.readerWait, -1) == 0 {
			runtime_Semrelease(&rw.writerSem, false)
		}
	}
	//[...]
}
```
首先将`readerCount`减去1，如果小于零，再讲`readWait`减去1，如果是离开读的`goroutine`数量为零，则对`writerSem`信号量进行`V`操作。

### 4.2.2 Lock和Unlock方法
```go
func (rw *RWMutex) Lock() {
    //[...]
	rw.w.Lock()
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_Semacquire(&rw.writerSem)
	}
	//[...]
}
```
首先`rw.w.Lock`操作，来防止其他`goroutine`对共享资源的写访问。然后将`readerCount`减去`rwmutexMaxReaders`，表明还剩多少`goroutine`可以进行读访问，这也解释在`RLock`中小于零的判断，如果还可以还可以进行读访问，则必须获得`readerSem`信号量。在`Lock`中接下来是对`readWait`判断，如果该数量不为零，则需要对`writerSem`进行`P`操作，而`V`操作只在`RUnlock`方法中，如果最后一个读`goroutine`离开，则进行`V`操作。

```go
func (rw *RWMutex) Unlock() {
    //[...]
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false)
	}
	//[...]
}
```
首先恢复`readCounter`为正数，然后对`readerSem`信号量进行`r`次`V`操作，唤醒在`RLock`中被挂起的`goroutine`。

## 4.3 sync.waitgroup.go
`WaitGroup`通常用在等待一组`goroutine`全部完成。调用`Add`方法指明要等待的`goroutine`的数量，调用`Done`方法说明该`goroutine`已经完成，而`Wait`方法是阻塞等待的`goroutine`。

### 4.3.1 数据结构
```go
type WaitGroup struct {
	noCopy noCopy
	state1 [12]byte
	sema   uint32
}
```
`noCopy`字段说明`WaitGroup`不允许拷贝，而`state1`字段是一个非常`tricky`的方法，用其中的`8`个字节(64bit)来保存一些状态。高位的32bit用来表示需要等待的`goroutine`的数量，地位的`32`bit用来表示被挂起的`goroutine`的数量。至于为什么不直接使用`64bit`的数据主要是为了考虑32为编译器无法保证64位对齐。`sema`则是一个信号量。
```go
func (wg *WaitGroup) state() *uint64 {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1))
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[4]))
	}
}
```
该方法是一个辅助方法，用来获取`state`，一个64为的无符号整数。

### 4.3.2 Add和Done方法
```go
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```
Done方法其实调用了`Add(-1)`方法，所以我们着重讨论`Add`方法。
```go
func (wg *WaitGroup) Add(delta int) {
	statep := wg.state()
	//[...]
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
			race.Read(unsafe.Pointer(&wg.sema))
	}
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	if v > 0 || w == 0 {
		return
	}
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// Reset waiters count to 0.
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(&wg.sema, false)
	}
}
```
首先是获取`state`，然后将`delta`右移32位，加上等待的`goroutine`数量。`v`和`w`分别代表了需要等待的`goroutine`和被阻塞的`goroutine`的数量。接下来`v==int32(delta)`判断条件表明如果是第一次`Add`操作，则必须与等待的`goroutine`同步，在`Wait`方法中可以看到同样的操作。接下来是一些抛异常操作，如果等待的数量为负数，如何第一次`Add`操作没有同步。`if >0 || w==0`条件表明如何`v`没有降到零，或者被阻塞的`goroutine`数量为零，直接返回。如何`v`为零，则按照`w`的数量，依次对信号量`ws.sema`进行`V`操作。

### 4.3.3 Wait方法
```go
func (wg *WaitGroup) Wait() {
    //[...]
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		w := uint32(state)
		//[...]
		// Increment waiters count.
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				race.Write(unsafe.Pointer(&wg.sema))
			}
			runtime_Semacquire(&wg.sema)
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			//[...]
			return
		}
	}
}
```
`Wait`方法同样也是CAS算法，首先获取需要等待的`goroutine`的数量`v`和阻塞的`goroutine`数量`w`, 然后将阻塞的`goroutine`数量+1，如果之前的`w`为零，表示是第一次等待，则与`Add`操作进行同步，最后后对信号量`wg.sema`进行`P`操作。

## 4.4 sync.cond.go
在编程中使用`Cond`也叫`管程(monitor)`，它可以用来使不同线程完成互斥条件，也可以使某个线程等待某个条件的发生。常见的使用模式如下： 
```go
var locker = new(sync.Mutex)
var cond = sync.NewCond(locker)
var condition = true
// goroutine A
cond.L.Lock()
for condition {
    cond.Wait()
}
// ...
cond.L.Unlock()

//goroutine B
condiiton = false
cond.Signal() 
```
为什么使用`for`循环作为判断进入`Wait`的条件而不是`if`呢？主要是防止为被唤醒的`goroutine`在返回`Wait`调用的时候，恰好有别的`goroutine`修改了`conditon`的值，所以需要使用`for`循环作为条件判断。
### 4.4.1 数据结构
```go
type Cond struct {
	noCopy noCopy
	L Locker
	notify  notifyList
	checker copyChecker
}
```
`Cond`结构不允许拷贝，包含了`Locker`的接口字段，和一个`notifyList`的集合字段。

### 4.4.2 NewCond函数
```go
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
```
实现`Locker`接口的类型都可以，一般为`Mutex`和`RWMutex`

### 4.4.3 Wait方法
```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```
在使用`Wait`方法之前，要调用`c.L.Lock`来进入临界区域，将当前等待的`goroutine`加入到通知队列中，然后调用`c.L.Unlock()`来退出临界区域，以便让其他`goroutine`可以进入等待区域。紧接着挂起`goroutine`，等待消息。
### 4.4.4 Singal方法
```go
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}
```
`runtime_notifyListNotifyOne`唤起其中的等待的`goroutine`。

### 4.4.5 Broadcast方法
```go
func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}
```
`runtime_notifyListNotifyAll`唤起全部等待的`goroutine`。
