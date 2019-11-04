---
date: 2019-11-03
status: public
tags: dotNet
title: coreFx 源码阅读
---

# 1 Collections

## 1.1 List

## 1.2 Hashtable

## 1.3 Dictionary

## 1.4 ConcurrentQueue

`ConcurrentQueue` 是线程安全的队列，用来实现最常见的生产者消费者模式，所以最重要的就是 `Enqueue` 和 `Dequeue` 接口。在 `coreFx` 中具体实现在 [ConcurrentQueueSegment.cs](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/Collections/Concurrent/ConcurrentQueueSegment.cs) 和 [ConcurrentQueue.cs](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/Collections/Concurrent/ConcurrentQueue.cs) 两个文件中。

在深入了解之前，我们先了解一下非线程安全的消息队列，一般使用 [`Circular Buffer`](https://en.wikipedia.org/wiki/Circular_buffer) 数据结构作为底层支持。

![CircularBuffer](./_image/circularBuffer.png)

如上图所示， `Circular Buffer` 其实就是一维数组和两个指针 `head` 和 `tail`，分别指向两端的操作，在 `Enqueue` 时候在 `tail+1` 位置插入，在 `Dequeue` 时候 `head+1`。同时为了实现循环的功能，每次 `Enqueue` 时候 `tail = (tail + 1) % Length`, 同样在 `Dequeue` 时候需要 `head = (head + 1) % Length`。在判断是否为空和大小的时候，也需要特殊处理。

```C#
int Count()
{
    (tail - head + Length) % Length + 1;
}

bool Empty()
{
    return tail == head;
}
```

虽然可以对 `Circular Buffer` 添加全局锁也可以实现线程安全的队列，但是限制了队列的长度大小，在需要扩容的时候需要申请更大的内存数组，效率上大大折扣。所以 `ConcurrentQueue` 将这些 `Circular Buffer` 链接起来。

![ConcurrentQueue](./_image/ConcurrentQueue.png)

**ConcurrentQueueSegment**

```C#
internal sealed class ConcurrentQueueSegment<T>
{
    internal readonly Slot[] _slots;

    internal PaddedHeadAndTail _headAndTail;
    //elide

    internal struct Slot
    {
        public T Item;

        public int SequenceNumber
    }

    //elide
}

internal struct PaddedHeadAndTail
{
    public int Head;

    public int Tail;
}
```

`_headAndTail` 包含 `Head` 和 `Tail` 两个指针，`_slots` 保存队列中的元素，除了 `Item` 保存元素，还包含 `SequenceNumber`，它用来表明当前 `slot` 是否可以 `Enqueue` 或者 `Dequeue`，规则如下：

- 对于往索引 `N` 位置插入一个元素，只有该位置的 `SequenceNumber` 值为 `N` 的时候才可以进行插入队列；
- 对于从索引 `N` 位置取出一个元素，只有该位置的 `SequenceNumber` 值为 `N+1` 的时候才可以进行取出；
- 当插入元素完成后，将该位置的 `SequenceNumber` 值为 `N+1`；
- 当取出元素完成后，将该位置的 `SequenceNumber` 值设置为 `N + _slots.Length`

在 `ConcurrentQueueSegment` 中最重要两个 API 接口分别为 `TryDequeue` 和 `TryEnqueue` ：

```C#
public bool TryDequeu(out T item)
{
    //elide
    var spinner = new SpinWai();
    while(true)
    {
        int current = Volatile.Read(ref _headAndTail.Head);
        int slotIndex = current & _slotMask;

        int sequenceNumber = Volatile.Read(ref slots[slotIndex].SequenceNumber);
        int diff = sequenceNumber - (currentHead + 1);
        if(diff == 0)
        {
            if(Interlock.CompareExchange(ref _headAndTail.Head, currentHead+1, currentHead) == currentHead)
            {
                item = slots[slotsIndex].Item;
                if(!Volatile.Read(ref _preserveForeObserveration))
                {
                    slots[slotsIndex].Item = default;
                    Valatile.Write(ref slots[slotsIndex].SequenceNumber, currentHead + slots.Length);
                }
            }
        }
        else if (diff <0)
        {
            bool frozen = _frozenForEnquences;
            int currentTail = Volatile.Read(ref _headAndTail.Tail)
            if(currentTail - currentHead <= 0 || (forzen && (currentTail - ForeezeOffSet - currentHead <=0)))
            {
                item = default!;
                return false;
            }
        }
        spinner.SpinOnce(-1);
        //elide
    }
}

public bool TryEnqueue(T item)
{
    //elide
    var spinner = new SpinWait();
    while(true)
    {
        int currentTail = Volatile.Read(ref _headAndTail.Tail);
        int slotsIndex = currentTail & _slotsMask;
        int sequenceNumber = Volatile.Read(ref slots[slotIndex].SequenceNumber);
        int diff = sequnenceNumber - currentTail;
        if (diff == 0)
        {
            if(Interlocker.CompareExchange(ref _headAndTail.Tail, currentTail=1, currentTail) == currentTail)
            {
                slots[slotsIndex].Item = item;
                Volatile.Write(ref slots[slotIndex].SequenceNumber, currentTail+1);
                reurn true;
            }
        }
        else if (diff < 0)
        {
            return false;
        }
    }
    spinner.SpinOnce(-1);
    // elide
}
```
为了线程安全，使用了 `SpinWait` 和 `CompareExchange` 控制并发逻辑。在 `TryDequeu` 中，计算待取出 slot 中的 `SequenceNumber` 和 `Head+1` 比较，如果差值等于 0，则取出该值；但是如果该队列被迭代之类，则不将该 slot 的 Item 设置为空。如果差值小于 0， 表明目前没有元素待取出，但是如果此刻有别的线程正在往其中插入数据，则自旋等待片刻，进入 while 下一次循环。`TryEnqueue` 同样如此，只不过当没有 slot 插入的时候，并没有等待取出线程完成，而是直接返回，因为队列插入过程是有序的。

`ConcurrentQueue<T>` 实现就比较简单，具体实现交给 `ConcurrentQueueSegment<T>`。

```C#
public class ConcurrentQueue<T>
{
    // elide
    private readonly object _crossSegmentLock;

    private volatile ConcurrentQueueSegment<T> _tail;

    private volatile ConcurrentQueueSegment<T> _head;

    public ConcurrentQueue()
    {
        _crossSegmentLock = new object();
        _tail = _head = new ConcurrentQueueSegment<T>(InitialSegmentLength);
    }
    // elide
}
```

`_tail` 和 `_head` 分别指向队列的首末的 `ConcurentQueueSegment` 片段，在初始化的时候，两者指向同一个 `Segment`，插入队列实现如下：

```C#
public void Enqueue(T item)
{
    if(!_tail.TryEnqueue(item))
    {
        EnqueueSlow(item);
    }
}
private void EnqueuSlow(T item)
{
    while(true)
    {
        ConcurrentQueueSegment<T> tail = _tail;
        if(tail.TryEnqueue(item))
        {
            return;
        }
        lock(_crossSegmentLock)
        {
            if(tail == _tail)
            {
                //elide
                int nextSize = tail._presevedForObserveraion ? InitialSegmentLength : Math.Min(tail.Capacity * 2, MaxSegmentLength);
                var newTail = new ConcurrentQueueSegment<T>(nextSize);
                tail._nextSegment = newTail;
                _tail = newTail;
            }
        }
    }
}

public bool TryDequeue(out T result) => _head.TryDequeue(out result) || TryDequeueSlow(out result);

private bool TryDequeueSlow(out T item)
{
    while(true)
    {
        ConccurrentQueueSegment<T> head = _head;
        if(head.TryDequeue(out item))
        {
            return true;
        }
        if(head._nexSegment == null)
        {
            item = default!;
            return false;
        }
        if(head.TryDequeue(out item))
        {
            return true;
        }
        lock(_crossSegmentLock)
        {
            if(head == _head)
            {
                _head = head._nextSegment;
            }
        }
    }
}
```

在插入的时候，首先在 `_tail` `segment` 中尝试插入，如果失败，就尝试创建新的 `Segment` 并且将 `_tail` 指针更新；在出队列的时候，同样先尝试从 `_head` 出队列，如果失败，就将 `_head` 指针指向下一个 `segment`，空闲的 `segment` 将会被 GC 处理。
还有一点要注意 `ConcurrentQueue` 的 `Count` 属性和 `IsEmpty` 属性时间效率是不一样的，`Count` 需要遍历全部的 `Segment`，而 `IsEmpty` 则尝试 `Peek` 队列。

(To be continued)
