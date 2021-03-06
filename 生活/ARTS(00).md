---
date: 2018-10-09
status: public
tags: ARTS
title: ARTS(00)
---
# 1 前言
ARTS是耗子叔([陈皓](https://coolshell.cn/haoel)）呼吁的一种学习方法，即每周做出如下的这些事情：
- Algorithm: 研究一个算法；
- Review：阅读一篇英文技术文章；
- Tip: 工作中的一个建议；
- Share: 分享一个观点。

# 2 Algorithm
**Sieve of Eratosthenes** (埃拉托斯特尼筛法)是古老的筛选质数的算法，该算法非常简单，主要流程如下：
1. 从2(最小的素数)开始的整数串，一直到`n`;
2. 首先令`p`等于`2`, 最小的素数；
3. 依次枚举`p`的整数倍，但不包括`p`，直到不大于`n`，将这些数据标记并过滤掉；
4. 在剩下的整数串中，选择比`p`大的最小数，并更新`p`为该值，重复步骤`3`；
5. 直到没有`p`值可以更新，算法结束，剩下的数据都是比`n`小的素数。
算法的正确性证明比较简单；
> 素数只能被`1`和本身整除，因此数据串中的所有素数都会保留下来；而合数(非质数)都可以分解质因数，所以所有的合数都会被筛选掉。

算法的时间复杂度: $O(n log logn)$， 但是算法需要一次次遍历整个数据串，如果`n`非常大，对内存的消耗也是非常大的。因此我们选择一种非常`tricky`的方式来处理：采用延迟生成的方式生成数据串，然后筛选掉每一个合数。
**Scheme** 版
`scheme`中提供了流(`Stream`)数据结构，该数据能够提供延迟计算的功能。
```scheme
;; 整除
(define (diviable? x y)
  (= (remainder x y) 0))
;; 整数流
(define (integers-start-from n)
  (cons-stream n (integers-start-from (+ n 1))))
;; 流过滤
(define (stream-filter pred stream)
  (if (stream-null? stream)
      the-empty-stream
      (if (pred (stream-car stream))
          (cons-stream (stream-car stream)
                       (stream-filter pred
                                      (stream-cdr stream)))
          (stream-fileter pred (stream-cdr stream)))))
;;筛选
(define (sieve stream)
  (cons-stream
   (stream-car stream)
   (sieve (stream-filter
           (lambda (x)
             (not (diviable? x (stream-car stream))))
           (stream-cdr stream)))))
;; 从2开始的素数流
(define primes (sieve (integers-start-from 2)))
```
**Golang**版
`go`语言中的通道(`channel`)也能提供类似的功能，一个通道负责生产整数串，其余的通道负责筛选数据。
```go
func generate(ch chan int){
    for i:=2; ; i++ {
        ch <- i 
    }
}
func filter(in, out chan int, prime int){
    for {
        i := <- in
        if i % prime !=0 {
            out<-i
        }
    }
}
func main(){
    ch := make(chan int)
    go generate(ch)
    for {
        prime := <- ch
        ch1 := make(chan int)
        go filter(ch, ch1, prime)
        ch = ch1
    }
}
```
# 3 Review
[使用Javascript函数式编程的体会](https://hackernoon.com/two-years-of-functional-programming-in-javascript-lessons-learned-1851667c726)
作者使用[Ramda](https://ramdajs.com/)和[Ramda-fantasy](https://github.com/ramda/ramda-fantasy)Javascript函数编程库编写了可视化在线IDE：[XOD project](https://xod.io/?utm_source=post&utm_campaign=fp_dev&utm_medium=medium)
得到的体会主要有一下几点：
1.  首先不局限于特定的编程语言，了解函数式编程的基本概念。但是大部分关于函数式编程的数据比较晦涩，推荐了几个比较容易的阅读材料。
    - [Mostly Adequate Guide to Functional Programming](https://mostly-adequate.gitbooks.io/mostly-adequate-guide/) 
    - [Think in Romda](http://randycoulman.com/blog/categories/thinking-in-ramda/)
    - [Professor Frisby Introduces Composable Functional JavaScript](https://egghead.io/courses/professor-frisby-introduces-composable-functional-javascript)
    - [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)
2. 函数式编程不仅仅关于`lambda calculus`，` monads`, `morphisms`, `combinators`,还有很多定义好的没有副作用组合函数。当然在使用之前你需要熟悉它们。
3. 使用`Maybes`和`Eithers`处理异常，它们能够帮助你理解函数式编程中的算子。
4. 选择好的函数式编程库和函数式编程的同事，它们能够帮助你一起成长。
5. 转换为函数式编程刚开始的时候非常不适应，如果克服了，将会受益匪浅。

# 4 Tip
在实际工作中，我们的代码可能依赖别人的代码，如果使用 *git* 管理我们代码仓库，将源代码直接拷贝到工作的仓库显然是不合理。那么如何进行管理引用的第三方的代码呢？答案就是: `git submodule`。
## 4.1 初始化submodule
为项目添加子模块
```shell
git submodule add https://github.com/gaufung/other-project.git
```
这样我们就会发现在项目文件夹下发现`other-project`文件夹和`.gitsubmodule`文件,文件内容为：
```
[submodule "other-project"]
	path = other-project
	url = https://github.com/gaufung/other-project.git
```
在`other-project`文件夹中做出的改动不会被我们仓库所记录，只有将目录`cd`到该目录下，所做的`git`操作才是针对该子仓库的操作。
## 4.2 克隆
如果克隆带有子模块的仓库，单单只克隆主仓库是无法获取子模块的目录是空的
```shell
git clone https://github.com/gaufung/main-project.git 
```
需要进行如下操作`git submodule init`和`git submodule update`. 
当然也有更方便的做法， 在克隆的时候需要添加`--recursive`参数即可。
# 5 Share
**成本**概念在经济学中与日常生活中有着不同的含义，当我们为了一件事情已经投入的时间、金钱和精力，在经济学中往往称为**沉没成本**，这些所谓的*成本*对我们接下来做决策没有实质影响，因此我们对*沉没成本*要做到及时止损，做出对当下和将来更好的决策。
当我们使用有限的资源做出一个决策的时候，往往也会放弃其他的一些决策，其他所有的决策的最大收益成为**机会成本**，*机会成本*告诉我们要全盘考虑，将资源使用在收益最大的地方。
日常生活中，我们考虑**成本**往往是是货币上的成本，但是时间，精力等因素都没有考虑，之所以思考方式是因为目前我们的时间和精力*不值钱*。因此我们首先要自己*值钱*起来，避免无效的时间浪费。