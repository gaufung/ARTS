---
date: 2019-04-29
status: public
tags: ARTS
title: ARTS(28)
---

# 1 Algorithm
>设计一个实时数据流接收器，要求在任意时刻输出所有数据的中位数。

初步设计在内部维护一个有序队列，每接受一个数据，插入到这个有序队列中，时间复杂度为 $O(n^2)$。既然需要输出数据流的中位数，那么只需要维护好这个中位数即可，至于其他数据可以降低排序的顺序要求，所以我们可以借助堆 (`heap`) 这个数据结构。
![](./_image/2019-04-29-19-26-18.jpg)
将整个数据流划分到这个两个堆中，数据流的前半部分将构建最大堆，而后半分布构建最小堆。为了保证快速获取数据的中位数，保证这个两个堆的数据量相差不超过 1， 那么中位数必定最大堆的顶元素和最小堆顶元素中产生（或者取平均数），所以在接受数据流的时候，往最大堆和最小堆中依次插入数据，在插入数据后，比较最大堆和最小堆的顶元素，如何最大堆的的顶元素比最小堆的顶元素大，则交换这两个顶元素，保证局部的偏序性。
上图为堆的逻辑视图，堆完全可以使用数组保存，按照堆的层次存放到数组中，存在下面的两个性质：
- $i$ 节点的父节点为$\lfloor \frac{n-1}{2} \rfloor $
- $i$ 节点的孩子节点为 $2 i + 1$ 和 $2i + 2$

初始化一个堆，从最后位置依次比较它和其父节点的大小，如果不满足顺序关系，交换数据，如此迭代直到第一个元素，时间复杂度为 $O(n)$。
将一个数据插入到堆中，首先将它插入到数组的最后，比较其和父节点顺序关系，如果不满足则交换数据，然后继续往上比较，类似冒泡的形式，时间复杂度为 $O(log(n))$。
将一个顶层元素从堆中删除，首先将最后一个元素置于顶层，然后分别比较其余两个子节点比较，如果不满足则与其中一个交换，重复如此下去，最后直达底层，时间复杂度为 $O(log(n))$
## 1.1 堆实现
```go
type Heap struct {
	elements []int
	compare  func(int, int) bool
}

func NewHeap(compare func(int, int) bool) *Heap {
	return &Heap{
		elements: make([]int, 0),
		compare:  compare,
	}
}

func (h *Heap) heap() {
	i := len(h.elements) - 1
	for ; i > 0; i-- {
		j := (i - 1) / 2
		if j >= 0 && h.compare(h.elements[i], h.elements[j]) {
			h.elements[i], h.elements[j] = h.elements[j], h.elements[i]
		}
	}

}

func (h *Heap) upHeap() {
	i := len(h.elements) - 1
	for i > 0 {
		j := (i - 1) / 2
		if h.compare(h.elements[i], h.elements[j]) {
			h.elements[i], h.elements[j] = h.elements[j], h.elements[i]
			i = j
		} else {
			break
		}
	}
}

func (h *Heap) downHeap() {
	size := len(h.elements)
	i := 0
	for i < size {
		left, right := 2*i+1, 2*i+2
		if right < size {
			index := right
			if h.compare(h.elements[left], h.elements[right]) {
				index = left
			}
			if h.compare(h.elements[index], h.elements[i]) {
				h.elements[index], h.elements[i] = h.elements[i], h.elements[index]
				i = index
			} else {
				break
			}
		}else if left < size {
			if h.compare(h.elements[left], h.elements[i]) {
				h.elements[left], h.elements[i] = h.elements[i], h.elements[left]
				i = left
				continue
			} else {
				break
			}
		}else{
			break
		}
	}
}

func (h *Heap) Top() int {
	return h.elements[0]
}
func (h *Heap) Pop() int {
	val := h.Top()
	h.elements[0], h.elements[len(h.elements)-1] = h.elements[len(h.elements)-1], h.elements[0]
	h.elements = h.elements[:len(h.elements)-1]
	h.downHeap()
	return val
}

func (h *Heap) Push(val int) {
	h.elements = append(h.elements, val)
	h.upHeap()
}

func (h *Heap) Empty() bool {
	return len(h.elements) == 0
}
```
## 1.2 中位数
```go
type MedianFinder struct {
	minHeap *Heap
	maxHeap *Heap
	cnt     int
}

/** initialize your data structure here. */
func Constructor() MedianFinder {
	min := func(a, b int) bool { return a < b }
	max := func(a, b int) bool { return a > b }
	finder := MedianFinder{cnt: 0}
	finder.minHeap = NewHeap(min)
	finder.maxHeap = NewHeap(max)
	return finder
}

func (this *MedianFinder) AddNum(num int) {
	if this.cnt%2 == 0 {
		this.maxHeap.Push(num)
	} else {
		this.minHeap.Push(num)
	}
	this.cnt = this.cnt + 1
	if this.minHeap.Empty() || this.maxHeap.Empty() {
		return
	}
	min := this.minHeap.Top()
	max := this.maxHeap.Top()
	if max > min {
		this.minHeap.Pop()
		this.minHeap.Push(max)
		this.maxHeap.Pop()
		this.maxHeap.Push(min)
	}
}

func (this *MedianFinder) FindMedian() float64 {
	if this.cnt%2 == 0 {
		return (float64(this.minHeap.Top()) + float64(this.maxHeap.Top())) * 0.5
	} else {
		return float64(this.maxHeap.Top())
	}
}
```
# 2 Review
[Two Star Programming](http://wordaligned.org/articles/two-star-programming)
**指针的指针编程**
几周之前，`Linus Torvalds`在 sloshdot 上回答了一些问题，他的所有回答都值得阅读，但是有一个吸引了我的注意。当被问到如何去描述他最喜欢内核取巧的地方，`Torvalds` 抱怨他最近很少看到的代码——除非是用来解决一些令人困扰的事情。他暂停了一下承认他非常自豪的内核在无所事事之前粗暴的查看文件名缓存。
> 与此相反，我事实上希望更多的人了解到底层代码真正的核心。并不复杂和事情比如无锁的名字查询，比如简单的使用指针的指针。举个例子，我看到好多人在删除单链表的条目的时候保留前面的条目，然后再删除这个条目，更下面的代码一样
```c
if (prev)
     prev->next = entry -> next;
else
    list_head = entry->next;
```
> 每当我看到这样的代码，我就认为 *这个人根本不懂指针*，不幸的是这个问题非常常见。
> 那些将指针只当作指向条目指针的人，通过 `list_head` 地址来初始化它们，然后遍历整个链表。他们移除条目而不需要任何条件，仅仅是通过 `*pp=entry->next` 方式完成。

我原本以为我理解指针，但是如果被如何去实现这个删除功能，我想我也会记录之前的节点，下面是简单的代码：
```c
typedef struct node
{
    struct node *next;
} node;
typedef bool ( * remove_fn) (node const * v) ;
//Remvoe all nodes form the supplied list for which the 
// supplied remove function return true.
// Returns the new head of the list
node * remove_if(node * head, remove_fn rm)
{
    for (node * prev = NULL, * curr =head; cur != NULL; )
    {
        node * const next = curr -> next;
        if (rm(curr) {
            if (prev)
                prev->next = next;
            else
                head = next;
        }
        else
            prev = curr;
        cur = next
    }
    return head
}
```
链表是非常简单的数据结构，通过构建指向下一个节点的指针和包含的数据构成。但是修改链表却非常微妙，所以在很多面试问题中，链表被广泛被问起。
上述的实现细微之处在于在处理待删除的节点的时候都都需要条件判断是否是头节点。
现在让我们看看 `Linus Torvalds` 建议的实现方式，在这里我们出入指向链表头结点的指针，然后遍历链表通过指向指针的下一个节点完成修改。
```c
void remove_if(node ** head, remove_fn rm)
{
    for (node ** cur = head; *cur; )
    {
        node * entry = *curr;
        if (rm(entry))
        {
            *curr = entry->next;
            free(entry);
        }
        else
            curr = & entry -> next;
    }
}
```
代码看上去好多了，关键点是链接至链表中的节点，也就是说指针的指针对于修改链表是更好的选择。
**译者注**
- [The mind behind Linux] (https://www.youtube.com/watch?v=o8NPllzkFhE) 在 TED 演讲中，`Linus Torvalds` 也描述过同样的问题。 
# 3 Tips

# 4 Share
**数据结构和算法**
- 数组能够保证 $O(1)$ 时间查询，而修改需要 $O(n)$ 时间复杂度；
- 数组的查找要求 $O(logN)$，则必须是二分查找，也就是每次判断必须要求丢弃近一半的数据
- 数组的区间表示方法 $[lo\space hi)$，左闭右开的表示，$hi-lo$ 则表示为区间的长度。
- 数组的基于比较的排序方法主要有
    - 冒泡排序（时间复杂度：$n^2$，稳定排序）
    - 插入排序  (时间复杂度：$n^2$，稳定排序)
    - 选择排序  (时间复杂度：$n^2$，不稳定排序)
    - 快速排序（时间复杂度： $nlogn$，不稳定排序）
    - 归并排序 （时间复杂度：$nlogn$, 稳定排序）
    - 堆排序（时间复杂度：$nlogn$，不稳定排序）
- 数组索引位置可以记录 $[0, i]$ 或者 $[i, n-1]$ 区间内的最值（或者其他值）
- 对数组先进行排序是处理问题的常用的方法
- 链表问题为了避免头指针的问题，可以引入指向头指针 `first` 指针，统一处理 corner case 问题
- 链表 `first` 和 `second` 指针可以在一次遍历中达到链表中的任意位置。
- 快慢指针通常是 `环检测` 的通用方法，也是链表环检测的方法。
- 树（二叉树）是一种严格递归形式定义的结构，一般来讲使用递归都能解决树相关的问题。
- 堆的逻辑视图是完全二叉树，物理视图为数组，堆的构建分为两种：一种是渐进式构建，一种是完全构建。
- 非递归的形式遍历二叉树，借助栈来模拟递归
- 图的 BFS 和 DFS 遍历
- 图 TS 排序
- 回溯算法，借助栈来实现（初始化的时候，首先插入 `哨兵` 值，可以统一整个回溯流程
- 动态规划问题，确定每一步所包含的状态（可能有多种状态），然后确定递推的公式
- 相同的值进行异或操作 `^`， 值为 0
不使用额外空间进行交换数据
```c
x = x ^ y
y = x ^ y
x = x ^ y
```
- 求众数问题，使用 `摩根算法` 筛选出候选者，再遍历一遍可以验证众数。
 