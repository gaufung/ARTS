---
date: 2019-02-24
status: public
tags: ARTS
title: ARTS(19)
---
# 1 Algorithm
> 给定一个整数数组 `nums` ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。
## 1.1 分析
假设我们得到最大和的连续子数组：$a_i+ a_{i+1} + \ldots + a_{j-1} + a_j$
根据最大和的要求，我们可以得出以下几点性质
- $a_i>0$，如果 $a_i <= 0$ 可以将这个子数组去掉，从而得到和更大的数组；
- $a_{i-1}<0$ 并且以 $a_{i-1}$结束的子数组和必定小于零，如果和大于零，那么可以将之前的子数组加上获取更大的子数组；
由此可以得出如下结论，$a_{i-1}$为数组的分界点，如果以$a_{i-1}$结尾的子数组的和小于零，这清空该子数组，以 $a_i$ 开始重新计算子数组。
每次子数组添加的时候，需要计算子数组之和，并实时更新。
```go
func maxSubArray(nums []int) int {
    sum, maxSum := 0, nums[0]
    for i:=0; i<len(nums); i++ {
        if sum >= 0 {
            sum += nums[i]
        }else{
            sum = nums[i];
        }
        if sum > maxSum {
            maxSum = sum
        }
    }
    return maxSum
}
```
# 2 Review
[Different ways of caching and maintaining cache consistency](https://blog.the-pans.com/different-ways-of-caching-in-distributed-system/)
**不同种类的缓存和维持缓存一致性**
`Phil Karlton`曾经说过
> 在计算机科学中有两大难题：缓存有效性和命名。

这句话还有很多其他的版本，我个人最喜欢的是`Jeff Atwood`的引用
> 在计算机科学中有两大难题：缓存有效性，命名和边界误差。 

正确的缓存非常困难，在很多分布式系统中和其他问题一样。第一眼看上去好像不是很难，我将会浏览一下在分布式系统中差将的缓存方式，它应该覆盖了你将会使用的基本缓存系统，特别要注意的是我将会专注于维持缓存一直性问题。
## 2.1 缓存和缓存一致性
在我们讨论不同的缓存方式的时候，需要对于缓存和缓存一致性有深刻的了解，尤其是缓存一致性是是一个大的概念。
在这我们这样定义缓存：
> 一个隔离的系统，它用来存储其包含的数据存储中包含的一部分数据视图。

请注意这是一个非常通用宽松的定义，它包含了你对缓存的全部理解，它保存了数据存储中一样的数据。它甚至包含了一些我们正常不会认为是缓存的东西，举例来讲，数据库展开的二级缓存。在我们的定义中，它也是缓存，所以保持缓存一致性是非常。
缓存是一致的，如果它满足
> 最终的键`k`对应的缓存中的值应该和其包含在数据存储中的键值`k`是一致的。

根据这个定义，如果缓存不包含任何东西，它肯定是缓存一致的，但是我们不会关注这个，因为这个一点用也没有。

## 2.2 为什么使用缓存
通常缓存是用来提高读写性能的，在这里性能可能是延迟，吞吐量或者是资源使用情况。通常情况下这些都是相关的，保护数据库通常是设计缓存的主要目的，你也可以说是解决性能问题。

## 2.3 不同种类的缓存

### 2.3.1 `Look-aside/demand-fill` 缓存

![](./_image/2019-03-02-09-16-11.jpg)
对于`look-aside`缓存，客户端在查询数据存储的时候首先查询缓存，如果命中，则返回缓存中的值，如果没有命中，则返回来自数据存储中的值。在这里，还没有涉及到如何填充缓存，只是说明了缓存如何被查询。但是通常是`demand-fill`, 在这里的意思是如果缓存没有被命中，客户端不仅仅从数据存储中获取值，而且将这个值放入到缓存中。通常`look-aside`缓存都是`demand-fill`缓存，但是也不是绝对的。
是不是很简单？然而简单的`Look-aside/demand-fill`缓存可能会出现永久的不一致问题，这个被很多人所忽视了。本质上来讲，是因为客户端在将值放回缓存的时候，这个值已经不新鲜了，具体的例子
- 客户端得到一个缓存不命中；
- 客户端从`DB`中获取键值为`A`；
-  其他人更新了`DB`数据库，并且设置为`B`也是缓存中的入口无效
- 客户端将值`A`写入到缓存中。

从这个时间点之后，客户端一直从缓存中获取到的值为`A`，而不是`B`，然而它才是最新的值。行不行这个取决于你的使用场景，也取决于缓存的入口是否包含`TTL`。但是这个在使用`look-aside/demand-fill`缓存的时候需要格外特别注意。
问题是可以解决的, `Memcache`使用`lease`来解决这个问题。本质上来讲，客户端就是在针对缓存做了一次`read-modify-write`的操作，而且没有任何基础性保护工作。在这个场景中，`read`是从缓存中读取，`modify`是从数据库中读取，`write`是写回到数据库中。一个简单的解决方案就是用`ticket`来代表在缓存读取时候的状态和缓存写入的时候的状态相比较。在`Memcache`中叫做`lease`, 在每一次缓存修改中是简单的计数器，所以在`read`的时候，它从`memcache`宿主机中获取`lease`，然后在写入的时候将`lease`传入。如果宿主机上的`lease`已经发生修改，则写入过程失败。现在我们重新看一下之前的例子
- 客户端缓存不命中，并且得到了一个`lease`值为`L0`;
- 客户端从`DB`中获取了键值为`A`;
- 其他人更新了`DB`数据库，并且设置为`B`也是缓存中的入口无效，并且设置`lease`为`L1`
- 客户端将值`A`写回缓存失败，因为`lease`不一致

### 2.3.2 `write-through/read-through`缓存

![](./_image/2019-03-02-10-15-52.jpg)
`Write-through`缓存意味着对于修改，客户端直接写入到缓存。然后缓存异步地写入到数据库中。至于读缓存没有任何其余的操作，客户端可以选择`look-aside`或者`read-through`。
`Read-through`缓存意味着对于读，客户端直接从缓存中读取。如果缓存不命中，缓存负责从数据存储中获取数据并填满，然后返回给客户端查询。它不涉及任何写的操作，客户端可以选择`demand-fill`或者`write-through`
下面是一张表

![](./_image/2019-03-02-10-22-13.jpg)
拥有`write-through`和`look-aside`缓存不常见，因为你既然你已经在客户端和数据存储中间建立了一个服务，它知道和如何和数据存储打交道，为什么不直接进行读写操作呢？但是换句话说，由于缓存大小的限制，`write-through`和`look-aside`缓存组合是对于缓存命中率提高取决于你的查询模式，比如大部分请求查询会在写入后立即执行，所以`write-through`和`look-aside`缓存组合能提供最好的缓存命中组合。
现在看一下`write-through`和`read-through`缓存的一致性问题。对于的那个问题，只要为`write`添加`update-lock`，为`read`添加`fill-lock`正确处理，读写同样的`key`就可以被序列化，而且缓存一直性问题就能解决。如果有多个缓存副本，它就是一个分布式问题，潜在的解决方案是有的。最直接的解决方案就是让多副本的缓存对于修改增加日志，对于更新都基于日志。日志服务的目的是保持序列化。它可以是`Kafka`或者`MySQL binlog`。 只要全局都遵循这个方案，重新播放事件是可行的。最终缓存一直性可以保证。

### 2.3.2 `Write-back / memory-only` 缓存
还有一类缓存，它们有数据丢失的风险。`write-back`缓存会在写入数据存储之前确认写操作，这个显然会导致数据丢失如果在两者中间发生`crash`. 这类缓存的使用场景是大量的吞吐量和`QPS`。但是它并不关系持久化或者一致性问题。带有持久化的`Redis`可以关掉它落地磁盘的选项。
# 3 Tips 
## 3.1 Python中时间格式化
`datetime.strptime(date_string, format)`中格式化的形式

DIRECTIVE | DESCRIPTION | EXAMPLE OUTPUT
---| --- | --- 
%a	 |  Weekday as locale’s abbreviated name.	| Sun, Mon, …, Sat (en_US) So, Mo, …, Sa (de_DE)
%A |	Weekday as locale’s full name.	| Sunday, Monday, …, Saturday (en_US) Sonntag, Montag, …, Samstag (de_DE)
%w	| Weekday as a decimal number, where 0 is Sunday and 6 is Saturday.	| 0, 1, 2, 3, 4, 5, 6
%d |	Day of the month as a zero-padded decimal number.	 | 01, 02, …, 31
%b |	Month as locale’s abbreviated name.	 | Jan, Feb, …, Dec (en_US) Jan, Feb, …, Dez (de_DE)
%B	| Month as locale’s full name.	| January, February, …, December (en_US) Januar, Februar, …, Dezember (de_DE)
%m	| Month as a zero-padded decimal number.	| 01, 02 … 12
%y	| Year without century as a zero-padded decimal number.	| 01, 02, … 99
%Y	| Year with century as a decimal number.	| 0001, 0002, … , 9999
%H	| Hour (24-hour clock) as a zero-padded decimal number.  | 01, 02, … , 23
%I	 | Hour (12-hour clock) as a zero-padded decimal number.	| 01, 02, … , 12
%p |	Locale’s equivalent of either AM or PM.	| AM, PM (en_US) am, pm (de_DE)
%M	| Minute as a zero-padded decimal number.	| 01, 02, … , 59
%S	| Second as a zero-padded decimal number.	| 01, 02, … , 59
%f	| Microsecond as a decimal number, zero-padded on the left.	| 000000, 000001, …, 999999 Not applicable with time module.
%z	| UTC offset in the form ±HHMM[SS] (empty string if the object is naive).	| (empty), +0000, -0400, +1030
%Z	| Time zone name (empty string if the object is naive).	| (empty), UTC, IST, CST
%j	 | Day of the year as a zero-padded decimal number.	| 001, 002, …, 366
%U |	Week number of the year  (Sunday as the first day of the week) as a zero padded decimal number.  All days in a new year preceding the first Sunday are considered to be in week 0.	| 00, 01, …, 53
%W	| Week number of the year (Monday as the first day of the week) as a decimal number. All days in a new year preceding the first Monday are considered to be in week 0.	| 00, 01, …, 53
%c |	Locale’s appropriate date and time representation.	| Tue Aug 16 21:30:00 1988 (en_US) Di 16 Aug 21:30:00 1988 (de_DE)
%x |	Locale’s appropriate date representation.	| 08/16/88 (None) 08/16/1988 (en_US) 16.08.1988 (de_DE)
%X	| Locale’s appropriate time representation.	| 21:30:00 (en_US) 21:30:00 (de_DE)
%%	| A literal ‘%’ character. |	%

## 3.2 正则匹配
在匹配某个特定的字符的时候，需要加上`^...$`，表达完整的匹配。
# 4 Share
## 4.1 为什么 `Go` 可以支持上百万的线程，而`Java`只支持成千上万个线程？

- ` JVM`使用的操作系统线程(`Operating System Threads `)，这会导致一下两个问题；
1.  OS线程中的栈大小为固定的，一般为`1M`，而 `Go` 采用的是动态栈，默认为`4KB`;
2. OS线程切换的时候，平均切换大概需要`10`微妙

- `Go`中多个`goroutine`运行在同一个操作系统线程上，通过语言的调度器，将活跃的`goroutine`运行在同一个操作系统线程上。

## 4.2 对于每一笔消费，都要值得注意的是，是否为一个资产。