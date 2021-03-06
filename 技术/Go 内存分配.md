---
title: Golang 内存分配
date: 2019-03-25
status: public
---

# 1 基础概念
内存分配是所有编程语言的重点考虑的内容，通常这些是由运行时（`runtime`) 进行管理，以此来达到内存池，可复用和避免碎片化等目的。在 `Golang`  内存分配算法基于 [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)。
## 1.1 基础结构
在 `src/runtime` 目录下，包含了全部的内存分配的相关结构，最主要的结构主要有 `mspan`, `mcache`, `mcentral` 和 `mheap`。
```go
type mspan struct {
    next     *mspan    // in a span linked list
	prev     *mspan    // in a span linked list
	start    pageID    // starting page number
	npages   uintptr   // number of pages in span
	freelist gclinkptr // list of free objects
	//....
}
```
`mspan` 是由地址连续的页组成，在 `Golang` 中每个页大小为 `8KB`
```go
_PageShift = 13
_PageSize = 1 << _PageShift
```
`mspan` 是一个双向链表，用以分配对象的 `object`, 按照 8  字节划分为 n 种, 比如大小为 24 的 `object` 可以存储 `17-24` 字节的对象。虽然会浪费一些空间，但是方便管理，并且避免的更大的碎片化。
按照对象的大小，全局分配了 67 中不同的大小类别，对于超出一定阈值的对象 `32KB`，当做大对象处理。
```go
_NumSizeClass = 67
_MaxSmallSize = 32 << 10 // 32KB
```
mheap 是内存的表示
```go
_MaxMHeapList   = 1 << (20 - _PageShift)  // 127
type mheap struct {
    free      [_MaxMHeapList]mspan // free lists of given length
	freelarge mspan                // free lists length >= _MaxMHeapList
	busy      [_MaxMHeapList]mspan // busy lists of large objects of given length
	busylarge mspan                // busy lists of large objects length >= _MaxMHeapList
	central [_NumSizeClasses] struct {
	    mcentral mcentral
	}
}
```
mcentral 是中间过程表示
```go
type mcentral struct {
	lock      mutex
	sizeclass int32
	nonempty  mspan // list of spans with a free object
	empty     mspan // list of spans with no free objects (or cached in an mcache)
}
```
mcache 对于每一个 `goroutine`, 这也是 `tcmalloc` 的核心高效的核心所在，每个`goroutine` 都拥有一个 `macache`, 避免了内存分配过程中的加锁，大大提高了效率。
```go
type mcache struct {
	alloc [_NumSizeClasses]*mspan // spans to allocate from
}
```
整体的示意图如下
![](./_image/2019-03-30-10-06-19.jpg)

 # 2 分配过程
## 2.1 初始化
在初始化阶段，预先保留了一段虚拟地址空间，空间被划分为三个区域
```
+-----------------+----------------------+----------------------------+
| spans 512MB  |    bitmap 32GB      |     area 512GB                 |
+-----------------+----------------------+----------------------------+
```
这些信息保存在 heap 中，
```go
type mheap struct {
    spans  **mspan
    spans_mapped uintptr
    
    bitmap uintptr
    bitmap_mapped uintptr
    
    area_start  uintptr
    area_used uintptr
    area_end uintptr
    area_reseverd bool
}
```
`mallocinit` 函数通过调用操作系统相关的 `mmap` 申请虚拟空间，在此期间也调用 `mHeap_Init` 函数初始化 `mheap` 中的各种 `mspan` 链表。

## 2.2 分配
所有的内存分配都会调用 `newobject` 方法
```go
func newobject(typ *_type) unsafe.Pointer {
	flags := uint32(0)
	if typ.kind&kindNoPointers != 0 {
		flags |= flagNoScan
	}
	return mallocgc(uintptr(typ.size), typ, flags)
}
```
`flagNoScan` 表明这个对象没有指针，因为接下来对于不包含指针的对象有单独的处理。接下来所有的工作交给了 `mallocgc` 函数
```go
func mallocgc(size uintptr, typ *_type, flags uint32) unsafe.Pointer{
    if size == 0 {
		return unsafe.Pointer(&zerobase)
	}
	if size <= maxSmallSize {
		if flags&flagNoScan != 0 && size < maxTinySize {
			off := c.tinyoffset
			if off+size <= maxTinySize && c.tiny != nil {
				// The object fits into existing tiny block.
				x = add(c.tiny, off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			s = c.alloc[tinySizeClass]
			v := s.freelist
			if v.ptr() == nil {
				systemstack(func() {
					mCache_Refill(c, tinySizeClass)
				})
				shouldhelpgc = true
				s = c.alloc[tinySizeClass]
				v = s.freelist
			}
			s.freelist = v.ptr().next
			s.ref++
			// prefetchnta offers best performance, see change list message.
			prefetchnta(uintptr(v.ptr().next))
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset {
				c.tiny = x
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			var sizeclass int8
			if size <= 1024-8 {
				sizeclass = size_to_class8[(size+7)>>3]
			} else {
				sizeclass = size_to_class128[(size-1024+127)>>7]
			}
			size = uintptr(class_to_size[sizeclass])
			s = c.alloc[sizeclass]
			v := s.freelist
			if v.ptr() == nil {
				systemstack(func() {
					mCache_Refill(c, int32(sizeclass))
				})
				shouldhelpgc = true
				s = c.alloc[sizeclass]
				v = s.freelist
			}
			s.freelist = v.ptr().next
			s.ref++
			// prefetchnta offers best performance, see change list message.
			prefetchnta(uintptr(v.ptr().next))
			x = unsafe.Pointer(v)
			if flags&flagNoZero == 0 {
				v.ptr().next = 0
				if size > 2*ptrSize && ((*[2]uintptr)(x))[1] != 0 {
					memclr(unsafe.Pointer(v), size)
				}
			}
		}
		c.local_cachealloc += size
	} else {
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			s = largeAlloc(size, uint32(flags))
		})
		x = unsafe.Pointer(uintptr(s.start << pageShift))
		size = uintptr(s.elemsize)
	}
}
```
首先是 `size == 0` 特殊处理，所以下面等式是成立的
```go
type T struct {}
a, b := new(T), new(T)
fmt.Sprintf("%p", a) == fmt.Spritf("%p", b) // true
```
整体思路是
- 微小对象（16），采用 cache.tiny object 分配
- 小对象（128）， 使用`cache.alloc[sizeclass].freelist` 分配
- 大对象直接从heap 中分配

小对象首先尝试从`cache.tiny + cache.off` 中分配，如果空间足够，则继续按照`size`， 增加 `cache.off` 来分配内存，否则从 `cache.alloc[tinySizeClass]` 也就是 `cache.alloc[2]` 中获取相应的 `mspan` 的 `freelist`， 如果 `cache.alloc[2]` 也没有空闲的`mspan`, 那就调用 `mCache_Refill` 方法从 `mcentral` 中获取。
对于普通的对象，首先根据对象大小计算出 `sizeclass`，然后从 `cache`  中获取相应的`freelist`，如果出现空间不够，调用 `mCache_Refill` 来完成空间申请。
最后大对象申请就比较简单了，直接调用 `largeAlloc` 函数从空间中申请即可。

## 2.3 扩张
如果 `mcahe` 中的空间不够了， 调用`mcache_Refill` 方法来扩张
```go
	s := c.alloc[sizeclass]
	// Get a new cached span from the central lists.
	s = mCentral_CacheSpan(&mheap_.central[sizeclass].mcentral)
	c.alloc[sizeclass] = s
	return s
```
用 `mCetneral_CacehSpan` 中获取新的 `mspan` 来替换原先的`c.alloc[sizeclass]` 中的 `mspan`。
过程很简单，首先遍历 `mcentral` 中的 `noempty` 链表，查看可以使用的`mspan`, 如果没有过从`empty`中遍历，获得那些已经清空的`mspan`。如果还是没有有用的`mspan`， 从`mCentral_Grow` 函数从 `heap` 中获取新的`mspan`, 然后将它添加到 `mcentral`中 `noempty` 链表中。
`mheap_Alloc` 方法是分配内存的核心函数，重点是 `mHeap_AllocSpanLocked` 函数从`heap` 对象中获取空间的`mspan`
```go
func mHeap_AllocSpanLocked(h *mheap, npage uintptr) *mspan {
	var s *mspan

	// Try in fixed-size lists up to max.
	for i := int(npage); i < len(h.free); i++ {
		if !mSpanList_IsEmpty(&h.free[i]) {
			s = h.free[i].next
			goto HaveSpan
		}
	}

	// Best fit in list of large spans.
	s = mHeap_AllocLarge(h, npage)
	if s == nil {
		if !mHeap_Grow(h, npage) {
			return nil
		}
		s = mHeap_AllocLarge(h, npage)
		if s == nil {
			return nil
		}
	}

HaveSpan:
	// Mark span in use.
	if s.state != _MSpanFree {
		throw("MHeap_AllocLocked - MSpan not free")
	}
	if s.npages < npage {
		throw("MHeap_AllocLocked - bad npages")
	}
	mSpanList_Remove(s)
	if s.next != nil || s.prev != nil {
		throw("still in list")
	}
	if s.npreleased > 0 {
		sysUsed((unsafe.Pointer)(s.start<<_PageShift), s.npages<<_PageShift)
		memstats.heap_released -= uint64(s.npreleased << _PageShift)
		s.npreleased = 0
	}

	if s.npages > npage {
		// Trim extra and put it back in the heap.
		t := (*mspan)(fixAlloc_Alloc(&h.spanalloc))
		mSpan_Init(t, s.start+pageID(npage), s.npages-npage)
		s.npages = npage
		p := uintptr(t.start)
		p -= (uintptr(unsafe.Pointer(h.arena_start)) >> _PageShift)
		if p > 0 {
			h_spans[p-1] = s
		}
		h_spans[p] = t
		h_spans[p+t.npages-1] = t
		t.needzero = s.needzero
		s.state = _MSpanStack // prevent coalescing with s
		t.state = _MSpanStack
		mHeap_FreeSpanLocked(h, t, false, false, s.unusedsince)
		s.state = _MSpanFree
	}
	s.unusedsince = 0

	p := uintptr(s.start)
	p -= (uintptr(unsafe.Pointer(h.arena_start)) >> _PageShift)
	for n := uintptr(0); n < npage; n++ {
		h_spans[p+n] = s
	}

	memstats.heap_inuse += uint64(npage << _PageShift)
	memstats.heap_idle -= uint64(npage << _PageShift)

	//println("spanalloc", hex(s.start<<_PageShift))
	if s.next != nil || s.prev != nil {
		throw("still in list")
	}
	return s
}
```
首先按照所需的`page`，获取空闲的`span`, 直到最大的固定页数的`span`，如果还是没有，从大于`127`的空闲链表中获取`span`, 实在不行就只能从操作系统申请至少`1MB`的空间。对于获取的页面重新裁剪，放回原来的空间中，注意还要合并左右的空闲的`span`。

# 3 回收过程
回收的源头是垃圾清理工作，以 `span` 为单位，通过对比 `bitmap` 里标记扫描标记，逐步将 ` object` 收归为 `span`， 最终上交给 `central` 和 `heap` 进行复用。
## 3.1 扫描mspan
```go
func mSpan_Sweep(s *mspan, preserve bool) bool {
	sstart := uintptr(s.start << _PageShift)
	for link := s.freelist; link.ptr() != nil; link = link.ptr().next {
		if uintptr(link) < sstart || s.limit <= uintptr(link) {
			// Free list is corrupted.
			dumpFreeList(s)
			throw("free list corrupted")
		}
		heapBitsForAddr(uintptr(link)).setMarkedNonAtomic()
	}
	size, n, _ := s.layout()
	heapBitsSweepSpan(s.base(), size, n, func(p uintptr) {
		// At this point we know that we are looking at garbage object
		// that needs to be collected.
		if debug.allocfreetrace != 0 {
			tracefree(unsafe.Pointer(p), size)
		}

		// Reset to allocated+noscan.
		if cl == 0 {
			// Free large span.
			if preserve {
				throw("can't preserve large span")
			}
			heapBitsForSpan(p).initSpan(s.layout())
			s.needzero = 1

			// important to set sweepgen before returning it to heap
			atomicstore(&s.sweepgen, sweepgen)

			// Free the span after heapBitsSweepSpan
			// returns, since it's not done with the span.
			freeToHeap = true
		} else {
			// Free small object.
			if size > 2*ptrSize {
				*(*uintptr)(unsafe.Pointer(p + ptrSize)) = uintptrMask & 0xdeaddeaddeaddead // mark as "needs to be zeroed"
			} else if size > ptrSize {
				*(*uintptr)(unsafe.Pointer(p + ptrSize)) = 0
			}
			if head.ptr() == nil {
				head = gclinkptr(p)
			} else {
				end.ptr().next = gclinkptr(p)
			}
			end = gclinkptr(p)
			end.ptr().next = gclinkptr(0x0bade5)
			nfree++
		}
	})
	if !freeToHeap && nfree == 0 {
		// The span must be in our exclusive ownership until we update sweepgen,
		// check for potential races.
		if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
			print("MSpan_Sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
			throw("MSpan_Sweep: bad span state after sweep")
		}
		atomicstore(&s.sweepgen, sweepgen)
	}
	if nfree > 0 {
		c.local_nsmallfree[cl] += uintptr(nfree)
		res = mCentral_FreeSpan(&mheap_.central[cl].mcentral, s, int32(nfree), head, end, preserve)
		// MCentral_FreeSpan updates sweepgen
	} else if freeToHeap {
		if debug.efence > 0 {
			s.limit = 0 // prevent mlookup from finding this span
			sysFault(unsafe.Pointer(uintptr(s.start<<_PageShift)), size)
		} else {
			mHeap_Free(&mheap_, s, 1)
		}
		c.local_nlargefree++
		c.local_largefree += size
		res = true
	}
	if trace.enabled {
		traceGCSweepDone()
	}
	return res
}
```
首先标记未被使用的 `object`, 然后对于大对象直接释放，对于小对象的`mspan`，收集不可达对象的计数，然后调用 `mCentral_FreeSpan` 收集 `mspan`。`mCentral_FreeSpan` 如果首先将其添加 `noempty` 链表中，然后调用 `mheap_Free`方法来释放。
## 3.2 定时扫描
在 `proc.go`中的 `main.main` 方法中有一个 `sysmon`方法
```go
func sysmon() {
	// If we go two minutes without a garbage collection, force one to run.
	forcegcperiod := int64(2 * 60 * 1e9)

	// If a heap span goes unused for 5 minutes after a garbage collection,
	// we hand it back to the operating system.
	scavengelimit := int64(5 * 60 * 1e9)

	lastscavenge := nanotime()
	nscavenge := 0

	// Make wake-up period small enough for the sampling to be correct.
	maxsleep := forcegcperiod / 2
	if scavengelimit < forcegcperiod {
		maxsleep = scavengelimit / 2
	}

	lasttrace := int64(0)
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
				// Need to decrement number of idle locked M's
				// (pretending that one more is running) before injectglist.
				// Otherwise it can lead to the following situation:
				// injectglist grabs all P's but before it starts M's to run the P's,
				// another M returns from syscall, finishes running its G,
				// observes that there is no work to do and no other running M's
				// and reports deadlock.
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
		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace*1000000) <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
	}
}
```
遍历 `mheap` 中的 free 数组和 `freelarge` 链表，获取空闲的地址空间，然后调用操作系统的`madvice` 方法，通知操作系统回收一块内存，但是这个并不是得到全部保证。
# 4 参考
- [Go 语言学习笔记](https://book.douban.com/subject/26832468/)
- [GoLang](https://github.com/golang/go)
