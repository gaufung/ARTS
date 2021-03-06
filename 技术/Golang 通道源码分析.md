---
date: 2019-04-07
status: public
title: Golang 通道源码分析
---
# 1 通道 (chan)
通道（channel）是 `Golang` 语言并发处理的一大特色，也是 `CSP` 理论的重要实现。在 `Golang` 中有一句非常著名的引用
> Don't communicate by sharing memory, sharing memory by communicating

那么通道和互斥锁 (Mutex) 在使用的时候有什么区别呢？
> 在业务层使用通道，在数据保护层使用互斥锁

通道分为两种，分为同步和异步，两者的区别在于是否用缓冲。
```go
func main(){
    c1 := make(chan int)
    c2 := make(chan int, 4)
}
```
上述中 `c1` 为同步通道， `c2` 为异步通道，不过都是使用同一个结构实现的。
```go
//src/runtime/chan.go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	lock     mutex
}
```
`dataqsiz` 为通道的大小，如果大于 0，表明是异步通道；`buf` 指向了通道传递的数据数组，也就是缓冲槽；`recvq` 和 `sendq` 分别指向了接收端和发送端的 `goroutine`。
`make(chan int)` 和 `make(chan int, 4)` 被转换成 `makechan` 函数:

```go
//src/runtime/chan.go
func makechan(t *chantype, size int64) *hchan {
	elem := t.elem
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	var c *hchan
	if elem.kind&kindNoPointers != 0 || size == 0 {
		c = (*hchan)(mallocgc(hchanSize+uintptr(size)*uintptr(elem.size), nil, flagNoScan))
		if size > 0 && elem.size != 0 {
			c.buf = add(unsafe.Pointer(c), hchanSize)
		} else {
			c.buf = unsafe.Pointer(c)
		}
	} else {
		c = new(hchan)
		c.buf = newarray(elem, uintptr(size))
	}
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	return c
}
```
首先会检查通道中数据类型的大小，如果大于 `64KB`，拒绝使用通道，因为这样数据拷贝非常占用内存；如果类型包含指针，则单独分配缓冲槽。
`recvq` 和 `sedq` 分别代表了收发 `goroutine`, 类型为 `waitq`
```go
//src/runtime/chan.go
type waitq struct {
	first *sudog
	last  *sudog
}

// src/runtime/runtime2.go
type sudog struct {
	g           *g
	selectdone  *uint32
	next        *sudog
	prev        *sudog
	elem        unsafe.Pointer // data element
	releasetime int64
	nrelease    int32  // -1 for acquire
	waitlink    *sudog // g.waiting list
}
```
这个 `sudog` 该怎么理解呢？对于接受的 `goroutine` 而言，当通道中没有数据或者缓冲槽已经满了，会将当前的 `g` 封装成 `sudog`，并且添加到当前 `hchan` 中的 `recvq` 链表中，并且将当前的 `g` 状态从 `running` 转换为 `waiting`，等待发送端的唤醒；同样的操作针对发送端，添加到 `sendq` 队列中，等待接收端的唤醒。
# 2 同步收发
## 2.1 同步发送
语法中的 `ch<-` 转换为 `chansend1` 函数
```go
//src/runtime/chan.go
func chansend1(t *chantype, c *hchan, elem unsafe.Pointer) {
	chansend(t, c, elem, true, getcallerpc(unsafe.Pointer(&t)))
}
func chansend(t *chantype, c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c.dataqsiz == 0 { // synchronous channel
		sg := c.recvq.dequeue()
		if sg != nil { // found a waiting receiver
			if raceenabled {
				racesync(c, sg)
			}
			unlock(&c.lock)

			recvg := sg.g
			if sg.elem != nil {
				syncsend(c, sg, ep)
			}
			recvg.param = unsafe.Pointer(sg)
			if sg.releasetime != 0 {
				sg.releasetime = cputicks()
			}
			goready(recvg, 3)
			return true
		}

		if !block {
			unlock(&c.lock)
			return false
		}

		// no receiver available: block on this channel.
		gp := getg()
		mysg := acquireSudog()
		mysg.releasetime = 0
		mysg.elem = ep
		mysg.waitlink = nil
		gp.waiting = mysg
		mysg.g = gp
		mysg.selectdone = nil
		gp.param = nil
		c.sendq.enqueue(mysg)
		goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

		// someone woke us up.
		gp.waiting = nil
		if gp.param == nil {
			if c.closed == 0 {
				throw("chansend: spurious wakeup")
			}
			panic("send on closed channel")
		}
		gp.param = nil
		releaseSudog(mysg)
		return true
	}
}
```
`chansend` 是通道发送的具体实现，如果 `c.dataqsiz` 为 `0`，表明是同步通道。
- 从 `recvq` 中获取一个 `sudog`
- 如果不为空，`syncsend()` 函数将发送端的 `ep` 拷贝到该 `sudog` 中的 `elem` 字段中，也就完成了通道传值的功能，对关联的 `g` 的 `param` 字段进行设值，以便接下来在唤醒这个 `g` 的时候，判断出是通道发送引起的，而不是 `close` 通道引起的。
- 如果没有发送端，那么将当前的 `g` 通过 `acquireSudog()` 方法创建一个 `sudog`, 然后设置 `elem` 等相关字段，将它添加到 `sendq` 队列中，并且调用 `goparkunlock()` 通知当前 `goroutine` 睡眠，也就是从 `running` 转态转换为 `waiting`  状态。
- 接下来如果通道数据被消费，那么这个 `g` 将会被唤醒，做一些清理的工作。

## 2.2 同步接受
语法中的 `<-ch` 转换为 `chanrecv1` 函数
```go
//src/runtime/chan.go
func chanrecv1(t *chantype, c *hchan, elem unsafe.Pointer) {
	chanrecv(t, c, elem, true)
}
func chanrecv(t *chantype, c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if c.dataqsiz == 0 { // synchronous channel
		sg := c.sendq.dequeue()
		if sg != nil {
			if ep != nil {
				typedmemmove(c.elemtype, ep, sg.elem)
			}
			sg.elem = nil
			gp := sg.g
			gp.param = unsafe.Pointer(sg)
			if sg.releasetime != 0 {
				sg.releasetime = cputicks()
			}
			goready(gp, 3)
			selected = true
			received = true
			return
		}
		// no sender available: block on this channel.
		gp := getg()
		mysg := acquireSudog()
		mysg.elem = ep
		mysg.waitlink = nil
		gp.waiting = mysg
		mysg.g = gp
		mysg.selectdone = nil
		gp.param = nil
		c.recvq.enqueue(mysg)
		goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

		// someone woke us up
		haveData := gp.param != nil
		gp.param = nil
		releaseSudog(mysg)

		if haveData {
			// a sender sent us some data. It already wrote to ep.
			selected = true
			received = true
			return
		}
	}
}
```
发送的过程和接受的过程正好相反
- 首先从`sendq` 中获取一个 `sudog`
- 如果有发送端，从中获取数据，并且唤醒被阻塞的发送端的 `g`
- 如果没有，则创建一个 `sudog` 然后添加到 `recvq` 队列中，将其进入睡眠
- 如果被唤醒，判断通过 `gp.param==nil` 判断是 `close` 唤醒还是发送端唤醒。 
- 返回发送端传递过来的数据。
# 3 异步收发
异步收发的方法和同步比较类似，只不过是通过 `qcount`, `sendx` 和 `recvx` 来进行阻塞的判断，而且数据传递也是通过这些计数器来确定。
## 3.1 异步发送
```go
//src/runtime/chan.go
func chansend(t *chantype, c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// asynchronous channel
	// wait for some space to write our data
	for futile := byte(0); c.qcount >= c.dataqsiz; futile = traceFutileWakeup {
		gp := getg()
		mysg := acquireSudog()
		mysg.releasetime = 0
		if t0 != 0 {
			mysg.releasetime = -1
		}
		mysg.g = gp
		mysg.elem = nil
		mysg.selectdone = nil
		c.sendq.enqueue(mysg)
		goparkunlock(&c.lock, "chan send", traceEvGoBlockSend|futile, 3)

		// someone woke us up - try again
		if mysg.releasetime > 0 {
			t1 = mysg.releasetime
		}
		releaseSudog(mysg)
		lock(&c.lock)
		if c.closed != 0 {
			unlock(&c.lock)
			panic("send on closed channel")
		}
	}

	// write our data into the channel buffer
   typedmemmove(c.elemtype, chanbuf(c, c.sendx), ep)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++

	// wake up a waiting receiver
	sg := c.recvq.dequeue()
	if sg != nil {
		recvg := sg.g
		unlock(&c.lock)
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		goready(recvg, 3)
	} else {
		unlock(&c.lock)
	}
	return true
}
```
- 当 `qcount` 大于 `dataqsiz`，说明当前的异步的通道已经退化为同步的通道，接下来的过程和之前一样；
- 如果缓冲区未满或者被发送端唤醒，那么使用`typedmemmove()` 方法将内容拷拷贝到 `buf` 中；
- 增加`sendx` 和 `qcount` 计数
- 从 `recvq` 获取一个 `sudog`, 如果不为空，唤醒这个 `g`

## 3.2 异步接受
```go
func chanrecv(t *chantype, c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
// asynchronous channel
	// wait for some data to appear
	var t1 int64
	for futile := byte(0); c.qcount <= 0; futile = traceFutileWakeup {
		if c.closed != 0 {
			selected, received = recvclosed(c, ep)
			if t1 > 0 {
				blockevent(t1-t0, 2)
			}
			return
		}

		if !block {
			unlock(&c.lock)
			return
		}

		// wait for someone to send an element
		gp := getg()
		mysg := acquireSudog()
		mysg.releasetime = 0
		if t0 != 0 {
			mysg.releasetime = -1
		}
		mysg.elem = nil
		mysg.g = gp
		mysg.selectdone = nil

		c.recvq.enqueue(mysg)
		goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv|futile, 3)

	}
	if ep != nil {
		typedmemmove(c.elemtype, ep, chanbuf(c, c.recvx))
	}
	memclr(chanbuf(c, c.recvx), uintptr(c.elemsize))

	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--

	// ping a sender now that there is space
	sg := c.sendq.dequeue()
	if sg != nil {
		gp := sg.g
		unlock(&c.lock)
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		goready(gp, 3)
	} else {
		unlock(&c.lock)
	}

	if t1 > 0 {
		blockevent(t1-t0, 2)
	}
	selected = true
	received = true
	return
}
```
过程几乎和异步发送相反
- 如果 `qcount <= 0`，没有数据缓冲槽中，封装成 `sudog` 然后添加到 `recvq` 队列中，并且休眠该 `g`
- 被唤醒，通过 `recvx` 读取缓冲槽中的数据
- 从 `sendq` 获取一个 `sudog`，如果有阻塞的 `sudog` ，唤醒对应的 `g`

# 4 关闭
`chan` 资源是不会被 gc 的， 所以在使用完 `chan` 后，必须要关闭整个 `channel`，语法也比较简单 `close(ch) ` 
```go
//src/runtime/chan.go
func closechan(c *hchan) {
	if c == nil {
		panic("close of nil channel")
	}
	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic("close of closed channel")
	}
	c.closed = 1
	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		gp := sg.g
		sg.elem = nil
		gp.param = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		goready(gp, 3)
	}

	// release all writers
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		gp := sg.g
		sg.elem = nil
		gp.param = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		goready(gp, 3)
	}
	unlock(&c.lock)
}
```
所以关闭 `nil` 或者已经关闭的 `chan` 会引发 `panic`。在设置完`closed` 字段后，遍历 `recvq` 和 `sendq` 队列，唤醒所有阻塞的 `g`
发送端和接受端对因为 `close` 而唤醒的处理方式也是不同的
```go
func chansend(t *chantype, c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

if c.closed != 0 {
		unlock(&c.lock)
		panic("send on closed channel")
}
}

func chanrecv(t *chantype, c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    if c.closed != 0 {
			return recvclosed(c, ep)
	}
}

func recvclosed(c *hchan, ep unsafe.Pointer) (selected, recevied bool) {
	if raceenabled {
		raceacquire(unsafe.Pointer(c))
	}
	unlock(&c.lock)
	if ep != nil {
		memclr(ep, uintptr(c.elemsize))
	}
	return true, false
}
```
对于发送给 `close` 的 `chan`， 会引发`panic`, 而接受来自`close` 的唤醒，`recvclosed` 会返回零值和`false`。

# 5 Select 语句
`select` 语句类似于 `linux` 中的 `select` 的功能，不同于 `IO` 复用，在 `golang` 中 `select` 主要用通道的收发。
## 5.1 使用方法
- **case  为空**
```go
func  main() {
    select {
        
    }
}
```
对于没有 `case` 的情况，阻塞当前 `goroutine`。

-  **只有一个case** 
```go
func main() {
    c1 : = make(chan int)
    select {
        case <- c1:
            print("c1")
    }
}

func main() {
    c1 : = make(chan int)
    select {
        case c1<- 1:
            print("c1")
    }
}
```
只有一个 `case` 的情况，`select` 将会退化成普通的通道的收发处理

-  **两个case，包含default**

```go
// receive
func main() {
    c1  := make(chan int)
    select {
        case <- c1:
                print("c1")
        default:
                print("default")
    }
}
// send
func main() {
    c2 := make(chan int)
    select {
        case c2 <- 1:
             print("c2")
         default:
             print("default")
    }
}
```
`default` 语句将 `select` 从阻塞状态转为为非阻塞状态，当 `c1` 或者 `c2` 通道阻塞的时候，会执行 `default` 语句，达到非阻塞效果。

- **多个case**
```go
func main() {
    c1, c2 := make(chan int), make(chan int)
    select {
        case <- c1:
            print("c1")
        case c2<-1:
            print("c2")
        default:
            print("default")
    }
}
```
这个是 `select` 中最为常见的使用方式，当 `c1` 和 `c2` case 均不可用额时候，就执行 `default` 语句。通常做法在最外面增加一层 `for` 循环，完成类似 *阻塞* 的功能。

## 5.2 源码分析

### 5.2.1 case 为空
```go
func main(){
    select {
        
    }
}
```
执行 `go test -o test test.go  & go tool objdump -s "main.main" test`  查看汇编代码
```asm
test.go:4             0x104ab4b               e8e04dfeff              CALL runtime.block(SB)
test.go:4             0x104ab50               0f0b                        UD2
```
通过汇编代码可以看出上述的代码将会转换为 `runtime.block`
```go
//src/runtime/select.go
func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1) // forever
}
```
`gopark` 将会让当前的 `goroutine` 永久休眠。

### 5.2.2 单个case
```go
func main(){
    c1 : = make(chan int)
    select {
        case <- c1:
            print("c1")
    }
}
```
查看汇编代码
```asm
test.go:8             0x108d6df               4885c0                  TESTQ AX, AX
test.go:8             0x108d6e2               745c                    JE 0x108d740
test.go:8             0x108d6e4               48890424                MOVQ AX, 0(SP)
test.go:8             0x108d6e8               48c744240800000000      MOVQ $0x0, 0x8(SP)
test.go:8             0x108d6f1               e8fa63f7ff              CALL runtime.chanrecv1(SB)
```
从汇编代码中可以看到， `select` 语句转为 `runtime.chanrecv1` 函数
```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}
```
这个 `chanrecv` 在之前的 `chan` 接受部分源码已经分析；同理如果是 `case` 只包含发送通道，同样调用 `chansend(c, elem, true, getcallerpc())` 方法。

### 5.2.3 单个case和default
```go
func main() {
    c1 := make(chan int)
    select {
        case <- c1:
            print("c1")
        default:
            print("default")
    }
}
```
查看汇编代码
```asm
test.go:8             0x108d6df               48c7042400000000        MOVQ $0x0, 0(SP)
test.go:8             0x108d6e7               4889442408              MOVQ AX, 0x8(SP)
test.go:8             0x108d6ec               e8af6cf7ff              CALL runtime.selectnbrecv(SB)
test.go:8             0x108d6f1               0fb6442410              MOVZX 0x10(SP), AX
test.go:8             0x108d6f6               84c0                    TESTL AL, AL
test.go:8             0x108d6f8               744a                    JE 0x108d744
test.go:9             0x108d6fa               0f57c0                  XORPS X0, X0
```
查看 `selectbrecv` 函数
```go
// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}
```
从注释可以看到编译器将上述的 `select` 形式转换为 `if ... else ...` 语句，而 `else` 分支就是 `default` 执行内容。从 `chanrecv()` 函数调用 `block` 参数为 `false`，也就是说当这个 `chan` 发送端没有 `sudog`，直接返回，而不是将这个 `g` 睡眠。
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool){
    //...
	if !block {
		unlock(&c.lock)
		return false, false
	}
	//...
}
```
同样的方式，如果 `select` 语句中的 `case` 只有发送通道和 `default` 语句，编译器会将其转换为下面函数
```go
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
```

### 5.2.4 多个case
```go 
func main() {
	c1, c2 := make(chan int), make(chan int, 3)
	select {
	case <-c1:
		print("c1")
	case c2 <- 2:
		print("c2")
	default:
		print("default")
	}
}
```
查看汇编代码
```asm
test.go:8             0x108f89b               488d442478              LEAQ 0x78(SP), AX
test.go:8             0x108f8a0               48890424                MOVQ AX, 0(SP)
test.go:8             0x108f8a4               e8e764faff              CALL runtime.selectgo(SB)
test.go:8             0x108f8a9               488b442408              MOVQ 0x8(SP), AX
```
从汇编中可以看出，对于`select` 的一般用法，将会调用 `selectgo` 方法
```go
//src/runtime/select.go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]
	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}
	// generate permuted order
	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	for i := 0; i < ncases; i++ {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := ncases - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}
	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

loop:
	// pass 1 - look for something already waiting
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c

		switch cas.kind {
		case caseNil:
			continue

		case caseRecv:
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
				goto rclose
			}

		case caseSend:
			if raceenabled {
				racereadpc(c.raceaddr(), cas.pc, chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}

		case caseDefault:
			dfli = casi
			dfl = cas
		}
	}

	if dfl != nil {
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		goto retc
	}

	// pass 2 - enqueue on all chans
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
		*nextp = sg
		nextp = &sg.waitlink

		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// wait for someone to wake us up
	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)

	sellock(scases, lockorder)

	gp.selectDone = 0
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
		} else {
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	if cas == nil {
		goto loop
	}

	c = cas.c

	if debugSelect {
		print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " kind=", cas.kind, "\n")
	}

	if cas.kind == caseRecv {
		recvOK = true
	}

	selunlock(scases, lockorder)
	goto retc

bufrecv:
	goto retc

bufsend:
	goto retc

recv:
	goto retc

rclose:
	goto retc

send:
	goto retc

retc:
	return casi, recvOK

sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```
- `pollorder` 和 `lockorder` 分别创建 case 乱序执行顺序和每个locker
- `for` 循序执行所有的 `case`, 通过 `pollorder` 随机选择待执行的case。
    - 对于同步接受通道，如果发送端有 `sudog`，则跳转到`recv` ，然后跳转到 `retc`; 对于异步接受通道，如果缓冲槽不为空，则跳转到`bufrecv`, 然后跳转到 `ret`;
    - 对于同步发送通道，如果接收端有 `sudog`，则跳转到 `send`，然后跳转到 `retc`; 对于异步发送通道，如果缓冲槽还有空，则条钻到`bufsend`, 然后跳转到 `retc`;
    - 如果是 `default`， 则设置 `dfli, dfl` 
- 在遍历完所有 case 之后，如果 `dfl` 不为空，表明命中了 `default`， 跳转到 `retc` 
- 将当前的 `g` 封装成 `sudog`, 分别添加到发送通道和接受通道中，休眠当前的 `goroutine`
- 一旦这个 `sudog` 被唤醒，遍历整个锁，判断被哪个通道唤醒，然后从其他通道删除这个`sudog`
- 最后跳转到 `retc`