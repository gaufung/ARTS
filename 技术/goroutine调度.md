---
title: Golang 并发调度
date: 2019-04-03
status: public
---

# 1 基础知识
`G` (goroutine), `M` (machine) 和 `P` (process) 是 goroutine 调度中最为核心的三个概念:
## 1.1 G
 代表了轻量级的线程，使用 `go fun()` 语句就可以创建一个 `goroutine`， 该结构主要包含的字段如下
```go
// src/runtime/runtime2.go
type g struct {
    stack       stack // 当前goroutine 使用的栈空间，注意栈从高地址向低地址扩展，[stack.lo, stack.hi).
    stackguard0 uintptr // 检查栈空间是否足够的值, 低于这个值会扩张栈, go 原生代码使用
    stackguard1 uintptr // 同stackguard0， 不过是原生代码使用
    m              *m     // 这个 g 当时使用的 m
    sched          gobuf // 当前g的调度数据，主要包含了pc和rsp等
    atomicstatus   uint32 // 当前g的状态
    schedlink      guintptr // 下一个g的链表
    preempt        bool    // 判断这个g 是否被抢占
    lockedm        *m // 判断这个g 是否锁定的特定的m
    //....
}
```
g 的状态主要有
- 空闲中(_Gidle): 表示G刚刚新建, 仍未初始化
- 待运行(_Grunnable): 表示G在运行队列中, 等待M取出并运行
- 运行中(_Grunning): 表示M正在运行这个G, 这时候M会拥有一个P
- 系统调用中(_Gsyscall): 表示M正在运行这个G发起的系统调用, 这时候M并不拥有P
- 等待中(_Gwaiting): 表示G在等待某些条件完成, 这时候G不在运行也不在运行队列中(可能在channel的等待队列中)
- 已中止(_Gdead): 表示G未被使用, 可能已执行完毕(并在freelist中等待下次复用)
- 栈复制中(_Gcopystack): 表示G正在获取一个新的栈空间并把原来的内容复制过去(用于防止GC扫描)

## 1.2 M
代表的系统线程(`OSThread`)， M可以运行两种代码
- go 代码，此时需要绑定一个 `P` 才能运行
- 系统调用代码，不需要 `P` 也能运行

该结构主要包含了字段如下
```go
// src/runtime/runtime2.go
type m struct {
    g0 *g // 用于调度的特殊 `g0`,
    curg          *g       // 当前运行的goroutine
    p             puintptr  // 运行时候包含的p
    nextp  puintptr // 即将获取的 p
    park          note // m 休眠信号量，通过此信号唤醒
    schedlink     muintptr // 下一个m
    mcache        *mcache // 本地缓存
    lockedg       *g // g 中 lockedm 字段对应值
}
```
m 也可以处于以下状态
- 自旋中(spinning): M正在从运行队列获取G, 这时候M会拥有一个P
- 执行go代码中: M正在执行go代码, 这时候M会拥有一个P
- 执行原生代码中: M正在执行原生代码或者阻塞的syscall, 这时M并不拥有P
- 休眠中: M发现无待运行的G时会进入休眠, 并添加到空闲M链表中, 这时M并不拥有P

## 1.3 P
代表 M 运行 G 时候需要的资源，默认为CPU核心数，通常代表了并行的程度，包含的主要的字段有
```go
// src/runtime/runtime2.go
type p struct {
	status      uint32 // p 当前所属的状态
	link        puintptr // 链表中下一个p
	m           muintptr // 当前 p 拥有的m
	mcache      *mcache
	runqhead uint32  // 本地运行队列的出队序号
	runqtail uint32 // 本地运行队列的入队序号 
	runq     [256]*g // 本地包含的goroutine 的列表，最多包含256 个
	runnext guintptr // 下一个要运行的goroutine

	// Available G's (status == Gdead)
	gfree    *g  //p 中包含空闲 g 的链表
	gfreecnt int32
     gcBgMarkWorker   *g // 后台进行 gc 的 g
}
```
P 也包含了相关的状态
- 空闲中(_Pidle): 当M发现无待运行的G时会进入休眠, 这时M拥有的P会变为空闲并加到空闲P链表中
- 运行中(_Prunning): 当M拥有了一个P后, 这个P的状态就会变为运行中, M运行G会使用这个P中的资源
- 系统调用中(_Psyscall): 当go调用原生代码, 原生代码又反过来调用go代码时, 使用的P会变为此状态
- GC停止中(_Pgcstop): 当gc停止了整个世界(STW)时, P会变为此状态
- 已中止(_Pdead): 当P的数量在运行时改变, 且数量减少时多余的P会变为此状态

## 1.4 协同工作
通过 `G`, `M` 和 `P` 三个抽象实体， 使得系统线程和用户线程(goroutine)形成了 `M:N`的映射运行关系
![](./_image/2019-04-04-11-47-41.jpg)
 上图为运行某一时刻的状态，每一个运行的 `M` 都对应于一个 `P`，并且运行相应的 `G`（图中蓝色的`G`)，同时每一个 `P` 也包含一个 `G` 队列，运行包含所有的 `goroutine`。

![](./_image/2019-04-04-13-28-14.jpg)
如果 `G0` 中包含了系统调用，那么将会获取或者新建一个 `M`(M1)，将 `G0` 单独与 `M0` 把绑定，原先的 `P` 与 `M1` 进行绑定执行。
# 2  源码分析
## 2.1 问题
在进入源码分析之前，带着相关的问题阅读源码
- g, m, p 是如何创建的？
- m 是如何绑定系统线程并且执行相应的 g ？
- g 中的栈是如何扩展和收缩的？
- 调度策略是怎样的？
- 如何对 g 进行抢占调度？

## 2.2 创建
### 2.2.1 创建 goroutine 
```go
package main
func add(x, y int) int {
    z := x + y
    return z
}
func main() {
    x := 0x100
    y := 0x200
    go add(x, y)
}
```
查看编译后的内容可以的得到
```asm
  test.go:11            0x104ab0d               48c744241000010000      MOVQ $0x100, 0x10(SP)  // 参数 x 入栈
  test.go:11            0x104ab16               48c744241800020000      MOVQ $0x200, 0x18(SP) // 参数 y 入栈
  test.go:11            0x104ab1f               c7042418000000          MOVL $0x18, 0(SP) // 参数长度 入栈
  test.go:11            0x104ab26               488d05bb000200          LEAQ go.func.*+58(SB), AX // 执行函数地址
  test.go:11            0x104ab2d               4889442408              MOVQ AX, 0x8(SP) // 函数地址入栈
  test.go:11            0x104ab32               e889f2fdff              CALL runtime.newproc(SB) // 调用 `runtime.newproc`
```
那么查看一下 `runtime.newproc` 方法
```go
// src/runtime/proc1.go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), ptrSize) 
	pc := getcallerpc(unsafe.Pointer(&siz))
	systemstack(func() {
		newproc1(fn, (*uint8)(argp), siz, 0, pc)
	})
}
```
`argp` 是第一个参数的位置，`pc` 为调用者的 PC 值，也就是 `main.main` 的 PC 值。准备好参数后然后调用 `newproc1` 方法
```go
// src/runtime/proc1.go
func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g {
	_g_ := getg()
	_g_.m.locks++ // disable preemption because it can be holding p in a local var
	siz := narg + nret
	siz = (siz + 7) &^ 7

	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_)
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	totalSize := 4*regSize + uintptr(siz) // extra space in case of reads slightly beyond frame
	totalSize += -totalSize & (spAlign - 1) // align to spAlign
	sp := newg.stack.hi - totalSize
	spArg := sp
	memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))
	memclr(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.sched.pc = funcPC(goexit) + _PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.startpc = fn.fn
	casgstatus(newg, _Gdead, _Grunnable)
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	runqput(_p_, newg, true)
	if atomicload(&sched.npidle) != 0 && atomicload(&sched.nmspinning) == 0 && unsafe.Pointer(fn.fn) != unsafe.Pointer(funcPC(main)) { // TODO: fast atomic
		wakep()
	}
	_g_.m.locks--
	if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
		_g_.stackguard0 = stackPreempt
	}
	return newg
}
```
这个函数就是创建 `goroutine` 的全部过程，从第三个参数在调用的实参是 `0`, 表明 `goroutine` 会丢弃掉返回值。大致的流程如下:
1. `getg` 获取当前运行的 `goroutine`；
2. `gfget(__p__)` 获得 `p` 中包含的空闲的 g；
3. 如果当前 `p` 没有空闲的 `g`, 那么通过 `malg()` 创建新的 `goroutine`；
4. 创建新的栈空间，并且将传过来的参数值进行拷贝；将 `newg` 的其他字段进行初始化；
5. `runqput` 将新创建的 `goroutine` 放置到队列中（可能是`p`的队列也有可能是全局队列）
6. 如果当前有个空闲的 `p` 并且没有 `m` 处于 `spining` 状态还有启动函数不是`main` 方法，调用 `wakeup` 来唤醒或者创建一个 `m` 来处理。

接下来看看`gfget()`函数如何去获取空闲的`goroutine`
```go
func gfget(_p_ *p) *g {
retry:
	gp := _p_.gfree
	if gp == nil && sched.gfree != nil {
		lock(&sched.gflock)
		for _p_.gfreecnt < 32 && sched.gfree != nil {
			_p_.gfreecnt++
			gp = sched.gfree
			sched.gfree = gp.schedlink.ptr()
			sched.ngfree--
			gp.schedlink.set(_p_.gfree)
			_p_.gfree = gp
		}
		unlock(&sched.gflock)
		goto retry
	}
	if gp != nil {
		_p_.gfree = gp.schedlink.ptr()
		_p_.gfreecnt--
		if gp.stack.lo == 0 {
			// Stack was deallocated in gfput.  Allocate a new one.
			systemstack(func() {
				gp.stack, gp.stkbar = stackalloc(_FixedStack)
			})
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gp.stackAlloc = _FixedStack
		} else {
			if raceenabled {
				racemalloc(unsafe.Pointer(gp.stack.lo), gp.stackAlloc)
			}
		}
	}
	return gp
}
```
`sched` 是一个全局变量，类型为 `schedt`，包含了全局的空闲的 `m`, `p` 和 `g`。
```go
//src/runtime/proc1.go
type schedt struct {
	lock mutex
	midle        muintptr // idle m's waiting for work
	nmidle       int32    // number of idle m's waiting for work
	nmidlelocked int32    // number of locked m's waiting for work
	mcount       int32    // number of m's that have been created
	maxmcount    int32    // maximum number of m's allowed (or die)
	pidle      puintptr // idle p's
	npidle     uint32
	nmspinning uint32
	// Global runnable queue.
	runqhead guintptr
	runqtail guintptr
	runqsize int32
	// Global cache of dead G's.
	gflock mutex
	gfree  *g
	ngfree int32
}
```
获取空闲的`g`的主要步骤如下
1. 如果 `p` 的队列为空，并且全局`sched` 队列不为空，从全局的队列中获取不超过32个 `g`
2. 再次从 `p` 的队列中获取 `g`，并且初始化栈大小
3. 如果实在找不到空闲的 `g`, 返回 `nil`，让调用者创建新的 `g`

创建新的 `g` 函数比较简单，使用`new` 创建 `g` 对象后，初始化其栈大小然后直接返回。
```go
//src/runtime/proc1.go
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(_StackSystem + stacksize)
		systemstack(func() {
			newg.stack, newg.stkbar = stackalloc(uint32(stacksize))
		})
		newg.stackguard0 = newg.stack.lo + _StackGuard
		newg.stackguard1 = ^uintptr(0)
		newg.stackAlloc = uintptr(stacksize)
	}
	return newg
}
```
`allgadd` 函数将所有创建的 `g` 添加到全局的数据中
```go
//src/runtime/proc.go
var (
	allgs    []*g
	allglock mutex
)
var (
    allg        **g
	allglen     uintptr
)
func allgadd(gp *g) {
	lock(&allglock)
	allgs = append(allgs, gp)
	allg = &allgs[0]
	allglen = uintptr(len(allgs))
	unlock(&allglock)
}
```
### 2.2.2 添加 g 至队列
在初始化 `g` 相关字段后，就可以将其放置到队列中，以方便 `M` 从中获取它们并执行。
```go
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand1()%2 == 0 {
		next = false
	}
	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	h := atomicload(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))] = gp
		atomicstore(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must suceed
	goto retry
}
```
在放置队列中的时候，如果想要将这个 `g` 放在队列中的下一个执行的位置，`next` 参数为`true`，那么新创建的 `g` 就会在下一次调度的时候首先执行。如果当前 `p` 的队列未满，将其放置在最后一个。否则调用 `runqputslow`放置到全局队列中。
```go
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
	var batch [len(_p_.runq)/2 + 1]*g
	n := t - h
	n = n / 2
	for i := uint32(0); i < n; i++ {
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))]
	}
	if !cas(&_p_.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
	batch[n] = gp

	if randomizeScheduler {
		for i := uint32(1); i <= n; i++ {
			j := fastrand1() % (i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	lock(&sched.lock)
	globrunqputbatch(batch[0], batch[n], int32(n+1))
	unlock(&sched.lock)
	return true
}
```
说明此时当前 `p` 已经满了， 从这个队列中获取一半的 `g` 加上新增加的 `g` 组成一个数组，然后将它们添加到全局队列中。

## 2.3 唤醒 M 执行
在创建一个 `g` 后，尝试调用 `wakeup` 来唤醒或者创建 `m` 来尽快执行
```go
//src/runtime/proc1.go
func wakep() {
	// be conservative about spinning threads
	if !cas(&sched.nmspinning, 0, 1) {
		return
	}
	startm(nil, true)
}
```
如果`sched.nmsping` 不为0， 也就是说存在 `m` 正在被唤醒，则返回，避免创建很多的线程。
```go
// src/runtime/proc1.go
func startm(_p_ *p, spinning bool) {
	lock(&sched.lock)
	if _p_ == nil {
		_p_ = pidleget()
		if _p_ == nil {
			unlock(&sched.lock)
			if spinning {
				xadd(&sched.nmspinning, -1)
			}
			return
		}
	}
	mp := mget()
	unlock(&sched.lock)
	if mp == nil {
		var fn func()
		if spinning {
			fn = mspinning
		}
		newm(fn, _p_)
		return
	}
	mp.spinning = spinning
	mp.nextp.set(_p_)
	notewakeup(&mp.park)
}
```
1. 首先判断是否有空闲的 `p`，如果没有则返回，因为如果没有空闲的 `p`, 则 `m` 无法绑定；
2. `mget()` 尝试获取空闲的 `m`, 如果没有空闲的 `m`，则 `newm()` 创建一个 `m`；
3. 将获取的空闲的 `m` 设置为 `spining` 状态，然后设置 `nextp` 为空闲的`p`；
4. `notewakeup()` 唤醒 `m`。

```go
func newm(fn func(), _p_ *p) {
	mp := allocm(_p_, fn)
	mp.nextp.set(_p_)
	msigsave(mp)
	newosproc(mp, unsafe.Pointer(mp.g0.stack.hi))
}

func allocm(_p_ *p, fn func()) *m {
	_g_ := getg()
	_g_.m.locks++ // disable GC because it can be called from sysmon
	if _g_.m.p == 0 {
		acquirep(_p_) // temporarily borrow p for mallocs in this function
	}
	mp := new(m)
	mp.mstartfn = fn
	mcommoninit(mp)
	if iscgo || GOOS == "solaris" || GOOS == "windows" || GOOS == "plan9" {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * stackGuardMultiplier)
	}
	mp.g0.m = mp
	if _p_ == _g_.m.p.ptr() {
		releasep()
	}
	_g_.m.locks--
	if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
		_g_.stackguard0 = stackPreempt
	}
	return mp
}
```
首先 `allocm` 创建一个新的 `m`，重点是初始化 `m.g0` 栈空间，然后调用 `newosproc()` 函数创建新的系统线程，注意并非所有的平台都支持 `m.g0` 栈空间。
```go
//src/runtime/os_linux1.go
var (
cloneFlags = _CLONE_VM | /* share memory */
		_CLONE_FS | /* share cwd, etc */
		_CLONE_FILES | /* share fd table */
		_CLONE_SIGHAND | /* share sig handler table */
		_CLONE_THREAD /* revisit - okay for now */
)
func newosproc(mp *m, stk unsafe.Pointer) {
	mp.tls[0] = uintptr(mp.id) // so 386 asm can find it
	var oset sigset
	rtsigprocmask(_SIG_SETMASK, &sigset_all, &oset, int32(unsafe.Sizeof(oset)))
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
	rtsigprocmask(_SIG_SETMASK, &oset, nil, int32(unsafe.Sizeof(oset)))
}
```
在这里最终要的是 `clone` 函数，针对 `Linux` 系统就是创建一个线程，`cloneFlags` 变量设置的该线程的相关属性。在执行运行时相关命令的时候，切换到`g0` 栈空间。在 `clone` 函数调用时候，将 `mstart` 函数作为启动函数传给你系统线程，那么现在看一下 `mstart` 是如果工作的。
```go
//src/runtime/proc1.go
func mstart() {
	_g_ := getg()

	if _g_.stack.lo == 0 {
		size := _g_.stack.hi
		if size == 0 {
			size = 8192 * stackGuardMultiplier
		}
		_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
		_g_.stack.lo = _g_.stack.hi - size + 1024
	}
	_g_.stackguard0 = _g_.stack.lo + _StackGuard
	_g_.stackguard1 = _g_.stackguard0
	mstart1()
}
```
首先初始化 `g0` 栈， 然后调用 `mstart1()` 函数
```go
//src/runtime/proc1.go
func mstart1() {
	_g_ := getg()
    gosave(&_g_.m.g0.sched)
	_g_.m.g0.sched.pc = ^uintptr(0) // make sure it is never used
	asminit()
	minit()
	if _g_.m == &m0 {
		// Create an extra M for callbacks on threads not created by Go.
		if iscgo && !cgoHasExtraM {
			cgoHasExtraM = true
			newextram()
		}
		initsig()
	}

	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}

	if _g_.m.helpgc != 0 {
		_g_.m.helpgc = 0
		stopm()
	} else if _g_.m != &m0 {
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	schedule()
}
```
首先 `gosave()` 函数保存 `g0` 相关设置，因为 `schedule()` 函数调用后，将会不重新设置 `g0`, 需要保存这个栈信息，以便其他 `g` 来调用。然后将这个 `m` 绑定一个`p` 而且将 `nextp`设置为 0，最后 `schedule()` 函数就是接下来整个调度的核心。
```go
//src/runtime/proc1.go
func schedule() {
	_g_ := getg()
	if _g_.m.lockedg != nil {
		stoplockedm()
		execute(_g_.m.lockedg, false) // Never returns.
	}

top:
	var gp *g
	var inheritTime bool
	if gp == nil && gcBlackenEnabled != 0 {
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		if gp != nil {
			resetspinning()
		}
	}
	if gp == nil {
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
			if gp != nil {
				resetspinning()
			}
		}
	}
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		if gp != nil && _g_.m.spinning {
			throw("schedule: spinning with local work")
		}
	}
	if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
		resetspinning()
	}

	if gp.lockedm != nil {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		startlockedm(gp)
		goto top
	}
	execute(gp, inheritTime)
}
```
这一部分的主要逻辑是获取一个 `g`， 然后调用 `execute()` 函数来执行这个 `g`，但是针对不同的 `g` 还做了特殊处理
- 如果这个 `g` 执行的 `m`的 `lockedg`字段不为空，则执行那么那个绑定的`g`
- 如果正在 `gc` 操作，取出要执行 `gc` 的 `g`， 然后执行
- 如果当前的 `p` 已经执行 62 次，就从全局队列中获取可执行的 `g`, 然后执行它，避免全局队列中 `g` 得不到执行的时刻
- 然后从 `p` 的队列中获取一个 `g`， 然后执行它
- 如果仍然没有可执行的 `g`, 然后调用 `findrunable()` 函数获取一个可执行的 `g`
- 对于获取的 `g`, 检查是否绑定了特定的 `m`，如果是，让绑定的 `m` 执行这个 `g`
- 最后调用 `execute()` 方法执行这个 `g`
```go
func execute(gp *g, inheritTime bool) {
	_g_ := getg()
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}
	_g_.m.curg = gp
	gp.m = _g_.m
	gogo(&gp.sched)
}
```
整个过程也比较简单
- 修改待执行的 `g`的状态，从 `runable` 到 `runing`
- 设置 `wait`, `preempt` 和  `statckguard0` 字段
- 如果有必要，增加这个 `p` 的调用次数
- 设置  `m.curg` 为要执行的 `g`，同时 `g` 和 `m` 为当前的 `m`
- 调用 `gogo()` 函数开始执行这个 `g`

`gogo` 是一个跟平台相关的函数，在 `amd64` 平台上，汇编代码如下
```asm
TEXT runtime·gogo(SB), NOSPLIT, $0-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		// make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	gobuf_sp(BX), SP	// restore SP
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP
	MOVQ	$0, gobuf_sp(BX)	// clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
	MOVQ	gobuf_pc(BX), BX
	JMP	BX
```
在前面我们分析过， `g.sched` 把保存栈执行上下文，在每次调用这个 `g` 时候，通过 `g.sched` 字段恢复上一次执行现场。在执行完毕后，将会调动 `goexit` 方法
```go
//src/runtime/proc1.go
func goexit1() {
	mcall(goexit0)
}

// goexit continuation on g0.
func goexit0(gp *g) {
	_g_ := getg()
	casgstatus(gp, _Grunning, _Gdead)
	gp.m = nil
	gp.lockedm = nil
	_g_.m.lockedg = nil
	gp.paniconfault = false
	gp._defer = nil // should be true already but just in case.
	gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
	gp.writebuf = nil
	gp.waitreason = ""
	gp.param = nil

	dropg()
	_g_.m.locked = 0
	gfput(_g_.m.p.ptr(), gp)
	schedule()
}
```
主要的工作是清理这个 `g` 对象，然后将其放回队列中以便复用这个 `g`，然后继续执行 `schedule()` 函数开始下一轮调度。在执行 `gfput()` 函数操作中，还做栈空间清理工作
```go
//src/runtime/proc1.go
func gfput(_p_ *p, gp *g) {
    stksize := gp.stackAlloc
	if stksize != _FixedStack {
		// non-standard stack size - free it.
		stackfree(gp.stack, gp.stackAlloc)
		gp.stack.lo = 0
		gp.stack.hi = 0
		gp.stackguard0 = 0
		gp.stkbar = nil
		gp.stkbarPos = 0
	} else {
		// Reset stack barriers.
		gp.stkbar = gp.stkbar[:0]
		gp.stkbarPos = 0
	}
	gp.schedlink.set(_p_.gfree)
	_p_.gfree = gp
	_p_.gfreecnt++
	if _p_.gfreecnt >= 64 {
		lock(&sched.gflock)
		for _p_.gfreecnt >= 32 {
			_p_.gfreecnt--
			gp = _p_.gfree
			_p_.gfree = gp.schedlink.ptr()
			gp.schedlink.set(sched.gfree)
			sched.gfree = gp
			sched.ngfree++
		}
		unlock(&sched.gflock)
	}
}
```
首先判断当前`g`栈空间是否超限，如果超限，释放它的栈空间。在将这个 `g` 放回绑定的 `p` 空闲队列是数量大于 64，那么将他们部分转移到全局队列中，本地队列最多保留 32 个空闲的 `g`。

## 2.4  获取可行执行的 g 
`findrunable()` 函数返回一个可行的 `g`, 为了尽最大可能快速执行，通过各种方法来获取 `g`
```go
// src/runtime/proc1.go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()
top:
	// local runq
	if gp, inheritTime := runqget(_g_.m.p.ptr()); gp != nil {
		return gp, inheritTime
	}

	// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_g_.m.p.ptr(), 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}
	if netpollinited() && sched.lastpoll != 0 {
		if gp := netpoll(false); gp != nil { // non-blocking
			// netpoll returns list of goroutines linked by schedlink.
			injectglist(gp.schedlink.ptr())
			casgstatus(gp, _Gwaiting, _Grunnable)
			return gp, false
		}
	}

	// If number of spinning M's >= number of busy P's, block.
	// This is necessary to prevent excessive CPU consumption
	// when GOMAXPROCS>>1 but the program parallelism is low.
	if !_g_.m.spinning && 2*atomicload(&sched.nmspinning) >= uint32(gomaxprocs)-atomicload(&sched.npidle) { // TODO: fast atomic
		goto stop
	}
	if !_g_.m.spinning {
		_g_.m.spinning = true
		xadd(&sched.nmspinning, 1)
	}
	// random steal from other P's
	for i := 0; i < int(4*gomaxprocs); i++ {
		if sched.gcwaiting != 0 {
			goto top
		}
		_p_ := allp[fastrand1()%uint32(gomaxprocs)]
		var gp *g
		if _p_ == _g_.m.p.ptr() {
			gp, _ = runqget(_p_)
		} else {
			stealRunNextG := i > 2*int(gomaxprocs) // first look for ready queues with more than 1 g
			gp = runqsteal(_g_.m.p.ptr(), _p_, stealRunNextG)
		}
		if gp != nil {
			return gp, false
		}
	}

stop:
	// We have nothing to do. If we're in the GC mark phase and can
	// safely scan and blacken objects, run idle-time marking
	// rather than give up the P.
	if _p_ := _g_.m.p.ptr(); gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != nil && gcMarkWorkAvailable(_p_) {
		_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
		gp := _p_.gcBgMarkWorker
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		return gp, false
	}

	// return P and block
	lock(&sched.lock)
	if sched.gcwaiting != 0 || _g_.m.p.ptr().runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
	if sched.runqsize != 0 {
		gp := globrunqget(_g_.m.p.ptr(), 0)
		unlock(&sched.lock)
		return gp, false
	}
	_p_ := releasep()
	pidleput(_p_)
	unlock(&sched.lock)
	if _g_.m.spinning {
		_g_.m.spinning = false
		xadd(&sched.nmspinning, -1)
	}

	// check all runqueues once again
	for i := 0; i < int(gomaxprocs); i++ {
		_p_ := allp[i]
		if _p_ != nil && !runqempty(_p_) {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				goto top
			}
			break
		}
	}

	// poll network
	if netpollinited() && xchg64(&sched.lastpoll, 0) != 0 {
		if _g_.m.p != 0 {
			throw("findrunnable: netpoll with p")
		}
		if _g_.m.spinning {
			throw("findrunnable: netpoll with spinning")
		}
		gp := netpoll(true) // block until new work is available
		atomicstore64(&sched.lastpoll, uint64(nanotime()))
		if gp != nil {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				injectglist(gp.schedlink.ptr())
				casgstatus(gp, _Gwaiting, _Grunnable)
				if trace.enabled {
					traceGoUnpark(gp, 0)
				}
				return gp, false
			}
			injectglist(gp)
		}
	}
	stopm()
	goto top
}
```
- `runqget()` 从本地队列中获取可执行的 `g`
- `globrunqget()` 从全局队列中获取可执行的 `g`
- `netpoll()` 从 `netpoll` 中获取可执行的 `g`
- 从其他 `p` 中获取可执行的 `g`

## 2.5 栈空间
`golang` 之所以能够很轻松的上万的 `goroutine`，一个主要的原因就是每个`goroutine` 的代价非常小，每个栈初始化大小 `2KB`, 而且在运行的过程中，还可以动态扩容，也可以动态收缩。通过上面你的分析也可以知道，`goroutine` 并不会被回收，这样对于栈空间管理就非常重要了。
首先栈的空间结构如下
![](./_image/2019-04-05-11-01-50.jpg)
`SP` 指针指向当前的栈顶， 从高地址向低地扩展。其中 `stackguard0`是非常重要的临界值，当 `SP` 值超出这个值，就会可能引发栈扩展操作。
```go
//src/runtime/stack2.go
const (
	// StackSystem is a number of additional bytes to add
	// to each stack below the usual guard area for OS-specific
	// purposes like signal handling. Used on Windows, Plan 9,
	// and Darwin/ARM because they do not use a separate stack.
	_StackSystem = goos_windows*512*ptrSize + goos_plan9*512 + goos_darwin*goarch_arm*1024

	// The minimum size of stack used by Go code
	_StackMin = 2048

	// The minimum stack size to allocate.
	// The hackery here rounds FixedStack0 up to a power of 2.
	_FixedStack0 = _StackMin + _StackSystem
	_FixedStack1 = _FixedStack0 - 1
	_FixedStack2 = _FixedStack1 | (_FixedStack1 >> 1)
	_FixedStack3 = _FixedStack2 | (_FixedStack2 >> 2)
	_FixedStack4 = _FixedStack3 | (_FixedStack3 >> 4)
	_FixedStack5 = _FixedStack4 | (_FixedStack4 >> 8)
	_FixedStack6 = _FixedStack5 | (_FixedStack5 >> 16)
	_FixedStack  = _FixedStack6 + 1 // now _FixedStack is power of 2 

	// Functions that need frames bigger than this use an extra
	// instruction to do the stack split check, to avoid overflow
	// in case SP - framesize wraps below zero.
	// This value can be no bigger than the size of the unmapped
	// space at zero.
	_StackBig = 4096

	// The stack guard is a pointer this many bytes above the
	// bottom of the stack.
	_StackGuard = 640*stackGuardMultiplier + _StackSystem

	// After a stack split check the SP is allowed to be this
	// many bytes below the stack guard.  This saves an instruction
	// in the checking sequence for tiny frames.
	_StackSmall = 128

	// The maximum number of bytes that a chain of NOSPLIT
	// functions can use.
	_StackLimit = _StackGuard - _StackSystem - _StackSmall
)
```
`_StackMin` 表明最小的栈大小为 `2KB`。对于不同的系统 `_StackSystem` 也是不同的。
### 2.5.1 栈初始化
```go
//src/runtime/proc1.go
func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g{
    if newg == nil {
		newg = malg(_StackMin)
	}
}
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(_StackSystem + stacksize)
		systemstack(func() {
			newg.stack, newg.stkbar = stackalloc(uint32(stacksize))
		})
		newg.stackguard0 = newg.stack.lo + _StackGuard
		newg.stackguard1 = ^uintptr(0)
		newg.stackAlloc = uintptr(stacksize)
	}
	return newg
}
```
`systemstack` 也就是 `g0` 栈空间分配栈空间后，立即设置 `stackguard0` 为 `lo+_StackGuard` 位置，`stackguard1` 设置为一个非常大值，因为这个用不到，`stackAlloc` 指示了当前的栈空间大小。

### 2.5.2 栈空间分配
`_FixedStack` 是一个分配的基础，不同系统按照不同的倍数进行放大
```go
type mcache struct {
    stackcache [_NumStackOrders]stackfreelist
}
type stackfreelist struct {
	list gclinkptr // linked list of free stacks
	size uintptr   // total size of stacks in list
}
```
对于不同的栈大小，都有对应的 `stackfreelist` 来分配空间，但是`_NumStackOrder` 对于不同的操作系统有一定的限制:
```
	// Number of orders that get caching.  Order 0 is FixedStack
	// and each successive order is twice as large.
	// We want to cache 2KB, 4KB, 8KB, and 16KB stacks.  Larger stacks
	// will be allocated directly.
	// Since FixedStack is different on different systems, we
	// must vary NumStackOrders to keep the same maximum cached size.
	//   OS               | FixedStack | NumStackOrders
	//   -----------------+------------+---------------
	//   linux/darwin/bsd | 2KB        | 4
	//   windows/32       | 4KB        | 3
	//   windows/64       | 8KB        | 2
	//   plan9            | 4KB        | 3
	_NumStackOrders = 4 - ptrSize/4*goos_windows - 1*goos_plan9
```
栈分配空间函数
```go
func stackalloc(n uint32) (stack, []stkbar) {
	var v unsafe.Pointer
	if stackCache != 0 && n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
		order := uint8(0)
		n2 := n
		for n2 > _FixedStack {
			order++
			n2 >>= 1
		}
		var x gclinkptr
		c := thisg.m.mcache
		if c == nil || thisg.m.preemptoff != "" || thisg.m.helpgc != 0 {
			lock(&stackpoolmu)
			x = stackpoolalloc(order)
			unlock(&stackpoolmu)
		} else {
			x = c.stackcache[order].list
			if x.ptr() == nil {
				stackcacherefill(c, order)
				x = c.stackcache[order].list
			}
			c.stackcache[order].list = x.ptr().next
			c.stackcache[order].size -= uintptr(n)
		}
		v = (unsafe.Pointer)(x)
	} else {
		s := mHeap_AllocStack(&mheap_, round(uintptr(n), _PageSize)>>_PageShift)
		v = (unsafe.Pointer)(s.start << _PageShift)
	}
    top := uintptr(n) - nstkbar
	stkbarSlice := slice{add(v, top), 0, maxstkbar}
	return stack{uintptr(v), uintptr(v) + top}, *(*[]stkbar)(unsafe.Pointer(&stkbarSlice))
}
```
整个过程非常简单，同内存分配比较类似，如果 `mcache` 的 `stackcache` 中仍然有空闲，分配给栈空间。如果没有空闲的栈，则调用 `stackcachefill` 方法进行填充，然后继续同样的工作。如果需要的栈空间超出 `stackcache` 大小限制，那么就调用`mHeap_AllocStack`在堆内存中分配。

### 2.5.3 栈空间扩容
在执行函数的时候，编译器会在函数头插入相关指令，来确定是否需要扩容。但是如果当前的 `SP` 小于 `stackguard0` 但是不超过 `_StackSmall` 的时候，是不需要扩容的
```go
//src/runtime/stack1.go
func newstack() {
	thisg := getg()
	gp := thisg.m.curg
	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0
	rewindmorestack(&gp.sched)
	// NOTE: stackguard0 may change underfoot, if another thread
	// is about to try to preempt gp. Read it just once and use that same
	// value now and below.
	preempt := atomicloaduintptr(&gp.stackguard0) == stackPreempt

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gwaiting)
	gp.waitreason = "stack growth"

	if gp.stack.lo == 0 {
		throw("missing stack in newstack")
	}
	sp := gp.sched.sp
	if thechar == '6' || thechar == '8' {
		// The call to morestack cost a word.
		sp -= ptrSize
	}
    casgstatus(gp, _Gwaiting, _Gcopystack)
	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
	copystack(gp, uintptr(newsize))
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}
```
通过`copystack` 函数，将当前的栈空间内容拷贝至新的 `stack` 中。在将 `g` 放回队列中的时候，也会调用 `stackfree` 释放空间，仅仅保留预留的空间大小。

## 2.6 调度
为了保证每个 `g` 都有执行的机会，所以需要给执行的 `g` 发出抢占的命令。`golang` 在这实现这一块的比较 `tricky`，并没有类似 CPU 时间片划分的概念。
在运行过程中 `sysmon` 监控 `goroutine` 一直执行，主要的工作的有下面几点
- 释放闲置5分钟的内存
- 如果超出2分钟没有发生垃圾回收，强制执行
- 长时间未处理的 `netpoll` 添加到任务队列中
- 向长时间运行的 `G` 发起抢占调度
- 回收因为 `sysCall` 长时间阻塞的回收 `P`
```go
//src/runtime/proc1.go
func sysmon() {
	// If we go two minutes without a garbage collection, force one to run.
	forcegcperiod := int64(2 * 60 * 1e9)

	// If a heap span goes unused for 5 minutes after a garbage collection,
	// we hand it back to the operating system.
	scavengelimit := int64(5 * 60 * 1e9)

	lastscavenge := nanotime()
	nscavenge := 0

	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)
	for {
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		usleep(delay)
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomicload(&sched.npidle) == uint32(gomaxprocs)) { // TODO: fast atomic
			lock(&sched.lock)
			if atomicload(&sched.gcwaiting) != 0 || atomicload(&sched.npidle) == uint32(gomaxprocs) {
				atomicstore(&sched.sysmonwait, 1)
				unlock(&sched.lock)
				notetsleep(&sched.sysmonnote, maxsleep)
				lock(&sched.lock)
				atomicstore(&sched.sysmonwait, 0)
				noteclear(&sched.sysmonnote)
				idle = 0
				delay = 20
			}
			unlock(&sched.lock)
		}
		// poll network if not polled for more than 10ms
		lastpoll := int64(atomicload64(&sched.lastpoll))
		now := nanotime()
		unixnow := unixnanotime()
		if lastpoll != 0 && lastpoll+10*1000*1000 < now {
			cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			gp := netpoll(false) // non-blocking - returns list of goroutines
			if gp != nil {
				incidlelocked(-1)
				injectglist(gp)
				incidlelocked(1)
			}
		}
		// retake P's blocked in syscalls
		// and preempt long running G's
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
		// check if we need to force a GC
		lastgc := int64(atomicload64(&memstats.last_gc))
		if lastgc != 0 && unixnow-lastgc > forcegcperiod && atomicload(&forcegc.idle) != 0 && atomicloaduint(&bggc.working) == 0 {
			lock(&forcegc.lock)
			forcegc.idle = 0
			forcegc.g.schedlink = 0
			injectglist(forcegc.g)
			unlock(&forcegc.lock)
		}
		// scavenge heap once in a while
		if lastscavenge+scavengelimit/2 < now {
			mHeap_Scavenge(int32(nscavenge), uint64(now), uint64(scavengelimit))
			lastscavenge = now
			nscavenge++
		}
	}
}
var pdesc [_MaxGomaxprocs]struct {
	schedtick   uint32
	schedwhen   int64
	syscalltick uint32
	syscallwhen int64
}
const forcePreemptNS = 10 * 1000 * 1000 // 10ms
```
`pdesc` 全局保存了全部 `p` 的执行的信息， `forcePreemptNS` 表示每隔10ms 开始发起抢占调度，由 `retake` 函数进行执行
```go
//src/runtime/proc1.go
func retake(now int64) uint32 {
	n := 0
	for i := int32(0); i < gomaxprocs; i++ {
		_p_ := allp[i]
		if _p_ == nil {
			continue
		}
		pd := &pdesc[i]
		s := _p_.status
		if s == _Psyscall {
			// Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
			t := int64(_p_.syscalltick)
			if int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
			// On the one hand we don't want to retake Ps if there is no other work to do,
			// but on the other hand we want to retake them eventually
			// because they can prevent the sysmon thread from deep sleep.
			if runqempty(_p_) && atomicload(&sched.nmspinning)+atomicload(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			// Need to decrement number of idle locked M's
			// (pretending that one more is running) before the CAS.
			// Otherwise the M from which we retake can exit the syscall,
			// increment nmidle and report deadlock.
			incidlelocked(-1)
			if cas(&_p_.status, s, _Pidle) {
				if trace.enabled {
					traceGoSysBlock(_p_)
					traceProcStop(_p_)
				}
				n++
				_p_.syscalltick++
				handoffp(_p_)
			}
			incidlelocked(1)
		} else if s == _Prunning {
			// Preempt G if it's running for too long.
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
				continue
			}
			if pd.schedwhen+forcePreemptNS > now {
				continue
			}
			preemptone(_p_)
		}
	}
	return uint32(n)
}
```
首先判断 `p` 当如果处于系统调用， 则判断执行时间，如果超出限制，则调用`handoffp()` 释放这个 `p`, 将 `g` 直接绑定到 `m`。接下来如果是 `p` 的状态正在执行，则调用 `preemtptone()` 放弃抢占调度。
```go
//src/runtime/proc1.go
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	if mp == nil || mp == getg().m {
		return false
	}
	gp := mp.curg
	if gp == nil || gp == mp.g0 {
		return false
	}
	gp.preempt = true

	// Every call in a go routine checks for stack overflow by
	// comparing the current stack pointer to gp->stackguard0.
	// Setting gp->stackguard0 to StackPreempt folds
	// preemption into the normal stack overflow check.
	gp.stackguard0 = stackPreempt
	return true
}
```
主要设置了两个字段：`preempt=true` 和 `stackguard0=stackPreempt`，因为 `g` 调用函数的使用，如果栈空间不够就会进扩容，而将`stackguard0`设置为`stackPreemt`将必然导致调用`newstack`方法
```go
//src/runtime/stack1.go
func newstack() {
	thisg := getg()
	preempt := atomicloaduintptr(&gp.stackguard0) == stackPreempt
	if preempt {
		if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
			// Let the goroutine keep running for now.
			// gp->preempt is set, so it will be preempted next time.
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gwaiting)
	gp.waitreason = "stack growth"
	if preempt {
		if gp.preemptscan {
			for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {
		}
			if !gp.gcscandone {
				scanstack(gp)
				gp.gcscandone = true
			}
			gp.preemptscan = false
			gp.preempt = false
			casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)
			casgstatus(gp, _Gwaiting, _Grunning)
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}

		// Act like goroutine called runtime.Gosched.
		casgstatus(gp, _Gwaiting, _Grunning)
		gopreempt_m(gp) // never return
	}
}
func gopreempt_m(gp *g) {
	goschedImpl(gp)
}
func goschedImpl(gp *g) {
	status := readgstatus(gp)
	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	lock(&sched.lock)
	globrunqput(gp)
	unlock(&sched.lock)
	schedule()
}
```
如果当前正在进行加锁操作，内存分配等过程，`gc` 扫描过程，不进行抢占。`goschedImpl` 将当前 `g` 状态设置为`runnable`, 然后将其放回 `golbrunqput` 放回全局队列中，最后 `schedule()` 开始下一路调度。

# 3 参考
- [Go语言学习笔记](https://book.douban.com/subject/26832468/)
- [Golang源码探索(二) 协程的实现原理](http://www.cnblogs.com/zkweb/p/7815600.html)