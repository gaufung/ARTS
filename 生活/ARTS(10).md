---
date: 2018-12-17
status: public
tags: ARTS
title: ARTS(10)
---

# 1 Algorithm
设计一个`LRU`缓存机制，使它支持`put`和`get`操作，缓存大小是由一定限制的，如果缓存数量超出范围，则删除最近最少使用的`key`，如果`key` 不存在，返回`-1`（所有的`value`值大于0），时间复杂度为$O(1)$。
为了保证时间复杂度为$O(1)$，就需要用空间换时间，为了保证更新`Key`和删除`Key`在常数时间内完成，所以采用双向链表完成，查询`Key`在常数时间内完成，则使用`字典`保存每一个`Key`，而且`Value`指向双向链表的每一个元素。依次`Put(1,1), Put(2,2), Put(3,3)`后，示意图如下:
![](./_image/2018-12-18-19-30-56.jpg)
`Front`节点为最近最少使用的`key`, `Back`节点为最近最多使用的节点。一旦`Get`命中，则将该节点中列表中摘除出来，并且添加到最后。
```go
type Element struct {
	key  int
	value int
	prev  *Element
	next  *Element
}

type List struct {
	head *Element
	tail *Element
}

func NewList() *List {
	l := &List{
		head:&Element{},
		tail:&Element{},
	}
	l.head.next = l.tail
	l.tail.prev = l.head
	return l
}

func (l *List) Remove(e *Element) {
	pre, next := e.prev, e.next
	pre.next = next
	next.prev = pre
}

func (l *List) Push(e *Element) {
	e.prev = l.tail.prev
	e.prev.next = e
	e.next = l.tail
	l.tail.prev = e
}

func (l *List) Pop() *Element {
	if l.head.next == l.tail {
		return nil
	}
	e := l.head.next
	l.Remove(e)
	return e
}

type LRUCache struct {
	l        *List
	capacity int
	table    map[int]*Element
}

func Constructor(capacity int) LRUCache {
	return LRUCache{
		l:        NewList(),
		capacity: capacity,
		table:    make(map[int]*Element),
	}
}

func (this *LRUCache) Get(key int) int {
	if val, ok := this.table[key]; ok {
		this.l.Remove(val)
		this.l.Push(val)
		return val.value
	}
	return -1
}

func (this *LRUCache) Put(key int, value int) {
	if val, ok := this.table[key]; ok {
		this.l.Remove(val)
		val.value = value
		this.l.Push(val)
	}else{
		e := &Element{key:key, value:value}
		if len(this.table) < this.capacity {
			this.l.Push(e)
			this.table[key] = e
		}else{
			n:=this.l.Pop()
			delete(this.table, n.key)
			n = nil
			this.l.Push(e)
			this.table[key] = e
		}
	}
}
```
# 2 Review
[Why I love refactoring](https://robertheaton.com/2014/07/05/why-i-love-refactoring/)
**为什么我喜欢重构**
我喜欢重构，如果不是显然的需求，这也是我软件开发的方法。可以甚至说，我喜欢重构超过喜欢写新需求。我几乎将圣诞假期、新年假期都用在在之前的代码中提取有用的东西。
当在重构的时候，你比当初写下第一行代码的时候更加了解这个领域，所以能够揭开无数之前的错误的假设。你也知道了这些代码在将来会怎么使用，因而更加有有序的组织它们。也知道性能方面问题是否存在，或者事实上性能是否你需要考虑的。
当你编写新的功能的时候， 代码库将会变得更加难以理解。你可能会编写一下很棒的代码，每一行的代码质量也会往上发展，但是这些都不重要的。我们都知道，大的代码库比小的代码库更加难以阅读，大代码库包含更多的文件夹，更多的文件和更多的层次设计，甚至都需要用`grep`工具来查询。但是如果使用重构，你将会往另外一个方向发展，将来的开发者在使用`git blame`的时候，会大大称赞你的名字。
还有更具有洞察力原因是重构有助于你的健康，但是这也不是我为什么喜欢重构的原因。我喜欢编写代码，既喜欢重构，也喜欢编写新的功能。但是没有比写出好的代码更加让人欣慰，没必要担心你的代码是否对别人有用，只要你自己知道就好了。你清楚地知道了使用哪种抽象，将你第一次编写功能的时候担心和优化隔离开来。
对于编写第一个版本的人保持尊重是非常重要的，他们也是聪明的人，是在你不同的限制条件下。或许联合国秘书长应该在将来24小时拍下一个快照，将来和现在都是不同的。没有人知道它的代码在将来会变得非常重要，甚至是在几年后还在运行着。他们的决定是重要的，你应该记住这一点。
不幸的是即使你有一个高品位的代码库，你也不会得到任何赞赏。但是实现它们是重要的而不是得到赞赏，要记住：让一些事变得有用起来。
# 3 Tips
# 3.1 github 更新`fork`的仓库
在`github`上参与开源项目，`fork`一份代码仓库后，如果原仓库有更新，那么如果更新自己`fork`出来的一份呢？
- 添加`remote`源: `git remote add upstream https://github.com/whoever/whatever.git`;
- 拉取源：`git fetch upstream`;
- 切换到`master`分支： `git checkout master`
- `rebase`分支：`git reabse upstream/master`
# 4 Share
单元测试一定要充分，现实往往是那些疏忽的单元测试出问题。