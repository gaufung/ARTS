---
date: 2019-06-17
status: public
tags: ARTS
title: ARTS(35)
---

# 1 Algorithm

> 设计一个数据结构，在下面的操作时间复杂度都是 $O(1)$
> - `insert(val)`: 当元素 `val` 不存在，向集合中插入该项。
> - `remove(val)`: 当元素 `val` 存在，从集合中删除该项。
> - `getRandom`: 随机返回集合中一项，每个元素都有相同的概率返回。

对于 `insert` 和 `remove` 操作，使用 `map` 数据结构可以很容易实现；但是 `getRandom` 操作不能在 $O(1)$ 时间复杂度完成，因此要借助数组才能满足条件。
![](./_image/2019-06-18-16-56-54.jpg?r=53)

上图为依次往集合中添加 `6、10、1、5、9` 元素，下面是数组，用来依次保存数据；上面是一个字典，`key` 为待插入元素，`value` 为该值在数组中的索引。


- insert 操作
插入元素首先在字典中查找，如果存在直接返回；否则在数组后面直接添加一个元素，并且在字典中添加相应的键值。

- getRandom 操作
随机获取元素比较简单，直接从数组中随机选择一个元素返回。

- remove 操作

删除操作就比较单独处理一下，如果要删除元素 `6`，现在字典中查找出索引为 `0`，然后与数组中最后一个元素 `9` 交换，然后更新字典中 `9` 对应的值为 `0`
![](./_image/2019-06-18-17-30-06.jpg?r=50)
现在待删除的元素 `6` 已经移动到数组末尾，现在就可以在 $O(1)$ 时间复杂度内删除数组和字典的词条。

```go
type RandomizedSet struct {
	values []int
	index  map[int]int
	size   int
}

/** Initialize your data structure here. */
func Constructor() RandomizedSet {
	return RandomizedSet{
		values: make([]int, 0),
		index:  make(map[int]int, 0),
	}
}

/** Inserts a value to the set. Returns true if the set did not already contain the specified element. */
func (this *RandomizedSet) Insert(val int) bool {
	if _, ok := this.index[val]; ok {
		return false
	} else {
		this.values = append(this.values, val)
		this.index[val] = len(this.values) - 1
		return true
	}
}

/** Removes a value from the set. Returns true if the set contained the specified element. */
func (this *RandomizedSet) Remove(val int) bool {
	if idx, ok := this.index[val]; ok {
		lastValue := this.values[len(this.values)-1]
		this.index[lastValue] = idx
		this.values[idx] = lastValue
		delete(this.index, val)
		this.values = this.values[:len(this.values)-1]
		return true
	} else {
		return false
	}
}

/** Get a random element from the set. */
func (this *RandomizedSet) GetRandom() int {
	idx := rand.Intn(len(this.values))
	return this.values[idx]

}
```
# 2 Review

# 3 Tips

# 4 Share