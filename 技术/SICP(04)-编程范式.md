---
title: SICP(04)-编程范式
status: public
date: 2018-10-07
---

# 1 命令式编程
## 1.1 局部状态值
当设计一个银行账户系统的时候，很自然的用一个变量`balance`保存账户余额，每次取款`withdraw`和存款`deposit`都更新账户余额，初步系统设计如下：
```scheme
(define  (make-account balance)
    (define (withdraw amount)
        (if (>= balance amount)
                (begin (set! balance (- balance amount))
                    balance)
                    "Insufficient funds"))
    (define (depoist amount)
        (set! balance (+ balance amount))
        balance)
   (define (dispath m )
        (cond ((eq? m 'withdraw) withdraw)
                  ((eq? m 'deposit) deposit)
                  (else (error ("Unknown request -- MAKE-ACCOUNT" m))))
dispatch)
```
在这里`set!`语句就是赋值更新语句, 而`balance`则是内部的账户的状态变量，通过更新状态变量来保证账户系统的功能。
虽然使用局部状态很好的描述问题，但是该计算模型也存在弊端。比如`(define W1 (make-account 100)`创建了账户`W1`，对账户进行两次操作`((W1 'withdraw) 20)`，账户余额分别为`80`和`60`，相同的输入没有生成相同的输出，这样对于`函数`的概念就需要重新审视；而且如果重新定义一个账户`(define W2 (make-account 100))`，那么`W1`和`W2`是“等价”的吗，也就是说可以替换掉吗？
上述的编程模型属于命令式编程(imperative programming)，通过赋值的方式修改变量通常使得我们需要考虑赋值的顺序，这一点是非常重要的。
## 1.2 并行缺陷
使用更新设值的方式还存在一个巨大的问题，也就是在并行化设计系统的时候会出现意想不到的问题。语句`(set! balance (- balance amount))`其实包含了三个步骤，
1. 取出`balance`变量
2. `balance`减去`amount`
3. 更新`balance`
如果多个用户并行操作该账户，将会产生意想不到的问题，在有关并行相关资料都会讨论到这一点。通常的解决方法是使用锁`Lock`机制，保证并发的有序执行。
# 2 流(Stream)编程
## 2.1 流数据结构
既然使用变量保存对象状态存在上述的弊端，我们引入一种流`Stream`数据结构来描述对象随着时间的变化的值。虽然流`Stream`结构看上去和之前的列表形式上很类似，但是存在一个非常重要的不同点：**延迟计算**。
熟悉`C#`中`LINQ`表达式或者`Spark`中的算子很好理解这个概念，延迟计算只表达了计算的分配而不是不会执行，只有当需要的时候才会引发计算。`Scheme`提供一系列针对`Stream`操作的方法:
- `(define z (cons-stream x y))` ：构建`stream:` `z`，包含两个部分`x`和`y`，其中`y`为延迟计算；
- `(stream-car z)`：返回`stream`第一个内容`x`;
- `(stream-cdr z)`: 执行`stream`第二部分内容，返回`y`.
除此之外还可以定义一些其他赋值方法来处理`Stream`结构。`(define (stream-ref s n))` 计算`s`前`n`个数据；`(define (stream-map proc . argstreams))` 对`argstreams`依次执行`proc`操作，注意这也是延迟操作的；`(define (stram-for-each proc s))`对`s`中每一个元素立即执行`proc`操作。

## 2.2 无穷流
既然流是按需计算元素，那么我们可以定义出一个无穷个元素流，而不用担心内存爆掉。实现无穷流，则采用递归的方式产生无穷流。

-  **整数流**
```scheme
(define (integers-starting-from n)
    (cons-stream n (integers-starting-from (+ n 1))))
(define integers (integers-starting-from 1))
```
-  **斐波那契数列**
```scheme
(define (fibgen a b)
    (cons-stream a (fibgen b (+ a b))))
(define fibs (fibs 0 1))
```
除了显示的生成方式生成无穷流，还可以隐式的方式定义流
- **1无穷流**
```scheme
(define ones (cons-stream 1 ones))
```
- **整数流**
```scheme
(define (add-streams s1 s2)
    (stream-map + s1 s2))
(define integers (cons-stream 1 (add-streams ones integers)))
```
- **斐波那契数列**
```scheme
(define fibs
    (cons-stream 0
        (cons-stream 1
            (add-streams (stream-cdr fibs
                        fibs))))
```
产生的数据流的示意图如下：
![](./_image/2018-10-08-19-30-46.jpg)

## 2.3 流计算模式
既然流避免的状态变量，那么可以通过流来替换掉原先计算模式，比如在先前计算平方根中，需要保存每次猜测的中间值，那么如果将每次迭代计算值作为流中的值，那么简化计算模式.
```scheme
(define (sqrt-improve guess x)
    (average guess (/ x guess)))
(define (sqrt-stream x)
    (define guesses
        (cons-stream 1.0
            (stream-map (lambda (guess)
                                        (sqrt-improve guess x)
                                    guesses)))
 guesses)
```
在使用流模式中，延迟计算起到关键的作用，通过延迟计算使得迭代计算可以定义出来。如果使用流模式，在先前的银行账户实例中，用户的每次的操作看作流中的元素，而余额则生成的流的内容，那么同样的输入则会生成相同的输出。