---
title: SICP(02)-过程抽象
status: public
date: 2018-09-17
---

# 1 Scheme 基础语法
`Scheme` 属于 `Lisp`的方言，都属于 [`S表达式`](https://zh.wikipedia.org/zh-hans/S-表达式)。如果熟悉编译原理的话，对于`C++`或者`JAVA`中的`1+ 6/2 ` 的表达式，在构建抽象语法树(Abstract Syntax Tree. AST)过程中，会根据操作数，操作符以及操作符之间的优先级构建出 ` (1+( 6/2) ` 的结构。但是在`Scheme`中则完全可以跳过这一步骤，因为其采用前缀表达式，使用括号将操作符所关联的操作数包含起来。那么上述的表达式可以如下表示`(+ 1 (/ 6 2))`, 这也是所说的`Code is data, data is code`，其弊端在于代码中括号噪声多，但是使用相关的插件都能够很好帮助组织代码。
## 1.1 表达式
$\frac{5+4+(2 - (3 - (6 + \frac{4}{5})}{3(6-2)(2-7)}$
```scheme
(/ (+ 5 
      4 
      (- 2 
         (- 3 
            (+ 6 
               (/ 4 5)))))
    (* 3 
       (- 6 2)
       (- 2 7))
   )
```
## 1.2 命名
使用`define`关键字进行命令
```scheme
(define size 2)
(define pi 3.1415926)
(define circumference (* 2 pi  10))
```
## 1.3 复合过程
自定义过程(`Procedure`), 具体的语法如下
```scheme
(define (<name> <formal parameter>)
<body>)
```
`<body>`也可以包含其他过程的定义，使之成为内部环境中的定义。
```scheme
(define (square x) (* x  x))
(square 2)
;; 4

(define (f a)
    (define (sum-of-square a b)
        (+ (square a) (square b)))
  (sum-of-square (+ a 1) (+ a 2)))
(f 2)
;; 25
```
## 1.4 条件表达式
`scheme` 中有两个条件表达式`cond`和`if`，使用方法分别如下
- `cond`

```scheme
(cond (<p1> <e1>)
          (<p2> <e2>)
          ...
          (<pn> <en>)
          (else <e>)
```
每一个预测分支`pi`如果为`true`则执行相应的表达式，如果一个预测条件没有达到则进入`else`分支中的表达式，注意`else`分支是可选的。
```scheme
(define (abs x)
    (cond ((> x 0) x)
              ((< x 0) (- x))
              (else 0)))
(abs 1)
;;1
(abs -2)
;;1
(abs 0)
;; 0
```
- `if`
```scheme
(if <predicate> <consequent> <alternative>)
```
如果`predicate`中为`true`，则执行`consequent`中语句，否则执行`alternative`语句。

## 1.4 布尔操作
布尔操作主要包含如下`not`, `and` 和 `or`，具体的操作逻辑和其他编程语言一样，而且对于`and`和`or`操作也存在 *短路效应*。

## 1.5 `let` 变量
使用`let`可以在过程中定义一些临时变量，方便代码书写， 语法如下：
```scheme
(let ((<var1> <exp1>)
      (<var2><exp2>)
        ...
      (<varn><expn>))
    <body>)
```

$f(x,y)=x(1+xy)^2 +y(1−y)+(1+xy)(1−y)$
```scheme
(define (f x y)
    (let ((a (+ 1 (* x y)))
           (b (- 1 y))
           (+ (* x (square a)) 
            (* y b)
            (* a b))))
```
## 1.6 `lambda` 表达式
`lambda`表达式通常用在高阶表达式中，定义一个匿名的过程作为其他过程的参数或者返回值，形式如下：
```scheme
(lambda (<formal-parameters>) <body>)
```
使用也非常简单
```scheme
(define (apply f x)
    (f x))
(apply (lambda (x) (* x x)) 2)
;; 4
```
# 2 过程抽象
函数就是过程抽象，类似于黑箱，函数使用者不关心过程的实现的细节，只关注函数名和相关参数。而且通过函数可以实现局部变量，做到环境隔离。接下来使用牛顿方法求解平方根
```scheme
(define (sqrt x)
    (define (good-enough? guess x)
        (< (abs (- (square guess) x))  0.001))
        (define (improve guess x)
            (average guess (/ x guess)))
        (define (sqrt-iter guess x)
            (if (good-enough? guess x)
                guess
                (sqrt-ite (improve guess x) x)))
    (sqrt-iter 1.0 x))
```
在这里 `sqrt` 函数屏蔽了内部定义的`good-enough?`、`improve`、`sqrt-iter`等函数，外部调用则不用关系内部具体实现过程，完成了抽象。
# 3 递归和迭代
## 3.1 递归
迭代和递归是计算机程序语言中最常见的形式，以阶乘来举例:
$$fractorial(n) = n * fractorial (n-1)$$
用递归函数的形式表示的话如下：
```scheme
(define (fractorial n )
    (if (= n 1)
        1
        (* n (fractorial (- n 1)))))
```
函数调用栈示意图如下
![](./_image/2018-09-22-09-40-35.jpg)
首先函数调用不停地调用自己，使调用栈原来越来越深，当抵达递归的出口，也就是`n=1`的时候，调用栈不停的释放，最后退出。使用递归的方式的好处是描述问题非常方便，也符合人的思考的方式，但是编写递归函数一定要考虑递归的出口。

## 3.2 迭代
如果我们的思维方式从调用栈最深的地方开始考虑问题，用`product`变量保存每一步的临时值
$$\begin{equation} \begin{cases} product \leftarrow count \times product  \\\ count  \leftarrow count + 1 \end{cases} \end{equation}$$
```scheme
(define (fractorial n )
(define (fact-iter product counter max-counter)
    (if (> counter max-count)
        product
        (fact-iter (* counter product)
                      (+ counter 1)
                     max-count))
 (fact-iter 1 1 n))
```
函数调用形式如下
![](./_image/2018-09-22-10-31-51.jpg)
## 3.3 递归计算过程和递归过程
如果说一个过程是递归的(`recursive procedure`)，说的是在语法形式直接或者间接地调用了自己。而对于一个递归计算过程(`recursive process`)，说的是采用递归的方式解决问题的的方式。对于`fact-iter`函数，虽然语法上调用自身，但是使用三个变量保存整个状态。在其他语言中，可以使用`while`，`for`等循环控制语句完成迭代。
# 4 形式抽象
既然函数能够做到抽象，然后能不能做到更高层次的抽象？ 当然能，在`Scheme`中函数作为`一等公民`（`first-class citizen`)，即它可以作为基础类型存在，既可以作为函数的参数，也可以作为返回的返回值存在。
对于数学上的求和公式
$$\Sigma_{i=a}^{b}f(i)=f(a)+\ldots+f(b)$$
对于上述的求和公式，我们可以得到更高层次的抽象，该抽象可以包含以下几个参数
- $f$：函数，签名接受一个参数并且返回一个数值；
- $next$: 函数， 给定一个参数返回下一个计算的值；
- $a$: 求和开始值；
- $b$: 求和结束值。
```scheme
(define (sum f a next b)
(if (> a b)
    0
    (+ (f a)
        (sum f (next a) next b))))
```
既然做到了加法求和的抽象，同样能不能做到乘法求机的抽象呢？答案是可以的，但是结合上述的情况，所以我们可以进行更高一层次的抽象，而刚刚的求和是该抽象的一种具体表现。
```scheme
(define (accumulate op init f a next b)
    (if (> a b)
        init
        (op (f a)
             (accumulate op init f (next a ) next b)))))
```
如何使用该抽象，对于使用阶乘的描述我们只需要关注该算子的每个参数即可，做到了模块化设计。
```scheme
(define (factorial n)
    (define (identify x) x)
    (define (inc x) (+ x 1))
    (accumulate * 1 identify 1 inc n))
```
关于函数级别的抽象先聊到这么多，接下来还有更多。
