---
date: 2019-03-04
status: public
tags: ARTS
title: ARTS(20)
---
# 1 Algorithm
> 在一个由 0 和 1 组成的二维矩阵内，找到只包含 1 的最大正方形，并返回其面积。
输入: 
1 0 1 0 0
1 0 1 1 1
1 1 1 1 1
1 0 0 1 0
输出: 4

采用动态规划的方法解决该问题，保存每个节点`(x,y)`作为正方形最右下角的最大边长。按行依次遍历所有的元素：
![](./_image/2019-03-07-11-39-15.jpg)
最大元素就是包含正方形的最大边长。

```go
func maximalSquare(matrix [][]byte) int {
    maxSide := 0
    sides := make([][]int, len(matrix))
    for i:= 0; i < len(matrix); i++ {
        sides[i] = make([]int, len(matrix[i]))
        for j:=0; j < len(matrix[i]); j++ {
            if i == 0 || j == 0 {
                sides[i][j] = toDigit(matrix[i][j])
                maxSide = max(maxSide, sides[i][j])
            }else{
                if matrix[i][j] == byte('0') {
                    sides[i][j] = 0
                    maxSide = max(maxSide, sides[i][j])
                }else{
                    leftValue := sides[i][j-1]
                    upperValue := sides[i-1][j]
                    upperLeftValue := sides[i-1][j-1]
                    minValue := minThree(leftValue, upperValue, upperLeftValue)
                    sides[i][j] = minValue + 1
                    maxSide = max(maxSide, sides[i][j])
                }
            }
        }
    }
    return int(maxSide * maxSide)
}

func max(a, b int) int {
    if a > b {
        return a
    }else{
        return b
    }
}

func toDigit(ch byte) int {
    return int(ch - byte('0'))
}

func minThree(a, b, c int) int{
    if a <= b && a <= c {
        return a
    }
    if b <= a && b <= c {
        return b
    }
    return c
}
```
# 2 Review
[An LRU in Go](https://roberto.selbach.ca/an-lru-in-go-part-1/)
**Go 实现 LRU 缓存**
在工作中，我常常发现需要使用一种方式来讲数据保存在内存中，以便我能够在接下来再一次使用它们。通常这就是缓存的目的，有时候也需要记录我之前想其他服务发送的消息以便我能够区别有什么不同，理由是多样的。
## 2.1 Naïve LRU
让我们开始最简单和最朴素的实现
```go
type LRU struct {
    cap int                           // the max number of items to hold
    idx map[interface{}]*list.Element // the index for our list
    l   *list.List                    // the actual list holding the data
}
```
在实现中，我们包含了一个双向链表 `l` 来保存数据，一个 `map` 来当索引；在 `Go` 中的通常做法，我们创建一个初始化器
```go
func New(cap int) *LRU {
    l := &LRU{
        cap: cap,
        l:   list.New(),
        idx: make(map[interface{}]*list.Element, cap+1),
    }
    return l
}
```
包使用者可以这样使用这个初始化器 `l := lru.New(100)` ，剩下的就非常简单了。但是我强烈坚信一个空的结构值也应该可以使用，所以下面的情况是不行的
```go
var l LRU
l.Add("hello", "world"")
```
这样会导致 `panic` 因为`l` 和 `idx` 都没有初始化。 我对于这种问题的解决方案是提供一个不导出的初始化函数，以便他们能够在每个导出方法开始的时候执行，确保初始化工作完成。
```go
func (l *LRU) lazyInit() {
    if l.l == nil {
        l.l = list.New()
        l.idx = make(map[interface{}]*list.Element, l.cap+1)
    }
}
```
现在让我们来看看如果在链表中加入一个元素
```go
func (l *LRU) Add(k, v interface{}) {
    l.lazyInit()

    // first let's see if we already have this key
    if le, ok := l.idx[k]; ok {
        // update the entry and move it to the front
        le.Value.(*entry).val = v
        l.l.MoveToFront(le)
        return
    }
    l.idx[k] = l.l.PushFront(&entry{key: k, val: v})

    if l.cap > 0 && l.l.Len() > l.cap {
        l.removeOldest()
    }
    return
}
```
在检查 `key` 是否已经在我们链表中之前， 先调用 `lazyInit` 方法来确保所有事情都已经初始化完毕。如果它在我们链表中，我们仅仅需要更新这个值，并且将它移动到链表之前，那么它现在就是最近使用的 `(most-mostly-used)`。
但是如果 `key` 不在我们的索引中，我们需要创建一个新的 `entry` 并且将它推入到链表中。顺便说一句，这是我们第一次看到 `entry`，这是一个不导出的类型，它通常保存在我们的的链表中
```go
type entry struct {
    key, val interface{}
}
```
最后，我们检查是否因为添加新的元素，导致链表的容量超出限制，如果超出限制，我们删掉链表中最老的元素。
这就是最朴素的实现，但是非常快：
>BenchmarkAdd/mostly_new-4                2000000            742 ns/op
BenchmarkAdd/mostly_existing-4          10000000            153 ns/op
BenchmarkGet/mostly_found-4             10000000            222 ns/op
BenchmarkGet/mostly_not_found-4         20000000            143 ns/op
BenchmarkRemove/mostly_found-4          10000000            246 ns/op
BenchmarkRemove/mostly_not_found-4      20000000            142 ns/op

获取和删除元素在同样的地方，即访问 `go` 内置 `map` 数据结构。获取一个已存在的对象需要一些额外的工作：将这个新的元素移动到列表最前面，但是对于指针操作也足够快。
添加的过程还需要检查链表的数量，删掉一些过期的数量。但是最大问题是这个实现方式不是并发安全的。当然，包的使用者可以在使用的时候用 `sync.Mutex` 来保护起来，但是我们希望将复杂度封装起来。
解决方案就是我们自己使用 `sync.Mutex` 将链表和 `map` 保护起来。

## 2.2 Concurrent-safe Naïve LRU
实现并发安全的并不需要改变很多，只需要添加一个锁
```go
type LRU struct {
    cap int // the max number of items to hold
    sync.Mutex                               // protects the idem and list
    idx        map[interface{}]*list.Element // the index for our list
    l          *list.List                    // the actual list holding the data
}
```
在导出的方法中，我们继续这样做
```go
func (l *LRU) Add(k, v interface{}) {
    l.Lock()
    defer l.Unlock()
    l.lazyInit()
```
完美，我们现在可以在高并发中使用，但是让我们看看性能测试报告
>BenchmarkAdd/mostly_new-4            2000000           819 ns/op
BenchmarkAdd/mostly_existing-4      10000000           200 ns/op
BenchmarkGet/mostly_found-4         10000000           277 ns/op
BenchmarkGet/mostly_not_found-4     10000000           219 ns/op
BenchmarkRemove/mostly_found-4       5000000           325 ns/op
BenchmarkRemove/mostly_not_found-4  10000000           220 ns/op

总体上来看，代码运行速度有点慢，因为过多的 `mutex` 操作。但是，尽管这个在并发中使用没有问题，但是没有真正使用它的优势。全部代码都是顺序的，`Add` 操作将会阻止任何人从中取回元素，甚至连 `Len()` 都会锁住全部。
对于并发测试，结果报告如下
> BenchmarkAddParallel/mostly_new-4            1000000          1040 ns/op
BenchmarkAddParallel/mostly_existing-4       5000000           433 ns/op
BenchmarkGetParallel/mostly_found-4          5000000           293 ns/op
BenchmarkGetParallel/mostly_not_found-4     10000000           289 ns/op
BenchmarkRemoveParallel/mostly_found-4       5000000           305 ns/op
BenchmarkRemoveParallel/mostly_not_found-4  10000000           291 ns/op

整体上来讲，调用过程需要花费更长的时间，因为大部分操作它们都需要等其他操作完成以便释放锁。
减少同步锁的时间并不实际的，你可以通过 `sync/atomic` 包完成指针移动而不适用锁机制。但是我们的情况比这个还复杂：我们拥有两个各自的数据结构来同步更新 (链表和索引)，所以不适用锁是无法做到的。
但是我们可以通过分片的方式来减少等待锁的时间。
现在正如你知道的，我们拥有单个的 `list` 来保存数据，一个 `map` 保存数据的索引。这也就意味着当一个 `goroutine` 想要进行添加操作 `{"foo": "bar"}`，它需要等待其他诸如 `{"bar": "baz"}` 更新完成， 因为尽管这两个 `key` 没有任何关联，但是锁需要保证全部的链表和索引。
一个快速的解决方案就是将整个数据拆分成多个分片，每个分片拥有自己的数据结构
```go
type shard struct {
cap int
len int32
sync.Mutex // protects the index and list
idx map[interface{}]*list.Element // the index for our list
l *list.List // the actual list holding the data
}
```
现在我们的 `LRU` 缓存有点不同
```go
type LRU struct {
    cap int // the max number of items to hold
    nshards int // number of shards
    shardMask int64 // mask used to select correct shard for key
    shards []*shard
}
```
那么 `LRU` 中的方法和它每个分片的方法调用是一样的
```go
func (l *LRU) Add(key, val interface{}) {
    l.shard(key).add(key, val)
}
```
所以现在是时候搞清楚如何为每一个 `key` 分配特定的分片。为了避免同一个的 `key` 的不同 `value` 被分配到不同的分片中，我们需要相同的 `key` 返回同一个分片。 我们可以通过将 `key` 通过 [Fowler-Noll-Vo hash function](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) 来生成一个 `int32` 的值，在哈希值与分片掩码通过逻辑 `AND`操作。
最难的部分就是获取一个 `key` 的字节表示，如果这个`key` 是固定类型，比如`int`或者`string`比较简单。但是事实上我们使用的是 `interface{}`，这个增加了我们的工作。`shard()` 函数事实上包含大的 `type switch`，它尝试用一种最快的方式来找到一个给定类型的字节表示形式。
举例来说，如果类型是 `int` 型的，我们可以这样做
```go
const il = strconv.IntSize / 8
func intBytes(i int) []byte {
    b := make([]byte, il)
    b[0] = byte(i)
    b[1] = byte(i >> 8)
    b[2] = byte(i >> 16)
    b[3] = byte(i >> 24)
    if il == 8 {
        b[4] = byte(i >> 32)
        b[5] = byte(i >> 40)
        b[6] = byte(i >> 48)
        b[7] = byte(i >> 56)
    }
    return b
}
```
对于所有已知的类型，都做同样的工作，包括对于提供了 `String() string`方法的自定义类型（即是想了 `Stringer` 接口）。这个包含了大部分用户想要使用的类型，然而对于未知类型或者没有个实现 `Stringer` 接口（或者 `Bytes() []byte`方法），我们就是用`gob.Encoder`方法，虽然很慢，但是足够使用。
有了上述的工作，现在我们不需要在操作的时候锁住全部的 `LRU`，而是仅仅锁住一种的一小部分，这就会导致在等待锁的时间消耗上大大减少。
分片的大小取决于我们的想要存储的数据的大小，但是我们可以拥有上千个小的的分片来提供很好的锁，当然由于处理小的分片有点过于设计。
下面是1000个分片的性能操作，没有并发执行
> BenchmarkAdd/mostly_new-4 1000000    1006 ns/op
BenchmarkAdd/mostly_existing-4 10000000  236 ns/op
BenchmarkGet/mostly_found-4 5000000  571 ns/op
BenchmarkGet/mostly_not_found-4 10000000     289 ns/op
BenchmarkRemove/mostly_found-4 3000000   396 ns/op
BenchmarkRemove/mostly_not_found-4 10000000  299 ns/op

下面是1000个分片包含的并发执行
>BenchmarkAddParallel/mostly_new-4 2000000    719 ns/op
BenchmarkAddParallel/mostly_existing-4 10000000  388 ns/op
BenchmarkGetParallel/mostly_found-4 10000000     147 ns/op
BenchmarkGetParallel/mostly_not_found-4 20000000     126 ns/op
BenchmarkRemoveParallel/mostly_found-4 10000000  142 ns/op
BenchmarkRemoveParallel/mostly_not_found-4 20000000  338 ns/op

[源代码](https://github.com/robteix/lru_blogpost/tree/master/sharded/lru)
# 3 Tips
- 在项目的解决方案设计中，所有的决策都需要以数据为准，哪怕采用封底估算的方式；
# 4 Share
## 4.1 关于中间件使用
在软件设计中，如果包含中间件，如果中间件出现问题，就需要在设计中包含
> 如果中间件出现问题，就一定保证程序正确的状态流转，也就是说保证程序的明确的输出。

## 4.2 关于缓存的设计
对于缓存之类的中间件，在设计中保证，缓存的中间出问题，只能降低软件的性能，而不能导致程序不能用。