---
title: SICP(03)-数据抽象
status: public
date: 2018-10-02
---
# 1 数据封装

## 1.1 封装
数据抽象的基本层次是将关联的数据组合在一起，比如有理数分为分子(`numerator`)和分母(`denominator`)，单个的分子和分母是没有意义的，因此需要一个构造器(`constructor`）:
```scheme
(define (make-rational (n d)
    (cons (n d)))
```
## 1.2 访问
但是在算术计算中，还是需要确定有理数的分子和分母，对于已经封装的对象，需要提供选择器(`selector`)来访问被封装的数据。
```scheme
(define (numerator r)
    (car r))
(define (denominator r)
    (cdr r))
```
## 1.3 接口
既然做到了数据的封装，就需要对封装的对象提供对外操作的接口，在接口中屏蔽操作的细节。对于有理数，最重要的就是算术运算，因此需要定义`add-rational`,`sub-rational`,`mul-rational`和`div-rational`运算
```scheme
(define (add-rational r1 r2)
    (make-rational (+ (* (numerator r1) (denominator r2))
                                 (* (numerator r1)(denominator r2)))
                             (* (denominator r1)(denominator r2))))
 (define (sub-rational r1 r2)
    (make-rational (- (* (numerator r1) (denominator r2))
                                 (* (numerator r1)(denominator r2)))
                             (* (denominator r1)(denominator r2))))
 (define (mul-rational r1 r2)
    (make ( (* (numerator r1) (numererator r2))
                (* (denomintor r1)(denomintor r2)))
(define (div-rational r1 r2)
    (make ( (* (numerator r1)(denominator r2)))
                (* (denominator r1)(numerator r1))))
```
通过数据抽象构造出如下的分层结构：

![](./_image/2018-10-02-10-30-15.jpg)
# 2 数据多种抽象
## 2.1 表示
有时候同一种对象有多种表示形式抽象表示，比如复数可以采用 $z=x + yi$ （`rectangular representation`） 或者 $z=re^{iA}$ (`polar representation`)，两种都有各自的优势，在计算复数的加法减法时第一种表示方式比较方便，而在计算乘法除法的时候，第二种方法比较方便。

![](./_image/2018-10-02-10-33-13.jpg)
## 2.2 公共接口
尽管有不同表示方式，但是每一种的实现都要实现如下的选择器`selectors`：`real-part`，`imag-part`，`magnitude`和 `angle`, 那么对于公共接口比如`add-complex`，`sub-complex`, `mul-complex`和`div-complex`就不需要关注复数实现的方式。

## 2.3 类型标签
但是除了保证选择器函数一致外，我们还可以使用为一个表示类型添加标签`tag`，在每一中数据类型抽象中构造器外再封装添加`tag`过程，这样每一个抽象类型可以自定义各个选择器的函数名，而不需要统一的接口。而在最外面提供选择器。
```scheme
;; ...
(define (make-from-real-imag-rectangular x y)
    (attach-tag 'rectangular (con x y)))
(define (make-from-mag-ang-polar  r a )
    (attach-tag 'rectangular (cons  (* r (cos a))
                                                      (* r (sin a))))
;;..
(define (make-from-real-imag-rectangular x y)
     (attach-tag 'polar (cons (sqrt (+ (square x)(square y)))
     (atan y x))))
 (define (make-from-mag-ang-polar r a)
    (attach-tag 'polar (cons r a)))
```
在为每一个复数类型添加了，可以首先判断其所属的类型而选择特定的选择器。
```scheme
(define (real-part z)
    (cond ((rectangular? z)
               (real-part-rectangular (contents z)))
               ((polar? z)
               (real-part-polar (contents z)))
;;..
```
## 2.3 数据导向
通过增加类型标签的方法虽然可以解决了接口统一的问题，但是存在以下缺陷
- 在开始设计之前必须知道全部表示形式，一旦增加或者减少类型，必须要修改公共接口；
- 不允许出现过程名相同。
可以采用数据导向型程序设计，假设存在如下的表格：
![](./_image/2018-10-02-21-34-56.jpg)
每一个过程都可以采用`Operation`和`Type`两个维度从二维表格中获取过程，而且每个类型都是独立的过程，互补影响。
```scheme
;; rectangular
(define (real-part z)(car z))
(put 'real-part '(rectangular) real-part)
;; polar 
(define (real-part z) (* (mangnitude z) (cos (angle z))))
(put 'real-part, '(ploar) real-part)
```
既然有了将过程放入表中，所以也要有将过程从表中取出并且执行的过程
```scheme
(define (apply-generic op . args)
    (let ((type-tags (map type-tag args)))
        (let ((proc (get op type-tag)))
            (if proc
                (apply proc (map contents args)))))
;;
(define (real-part z)(apply-generic 'real-part z))
```
采用数据导向设计方式，避免了标签设计的问题，增加数据抽象表达方式也不会修改原先设计。

## 3.4 消息传递
最后也可以通过类似消息传递的方式来进行过程分派。
```scheme
(define (make-from-real-image x y)
    (define (dispath op)
        (cond ((eq? op 'real-part) x)
                  ((eq? op 'imag-part) y)
                  ((eq? op 'magnitude)
                    (sqrt (+ (square x)(square y))))
                ((eq? op 'angle)(atan y x))
                (else "Unknown op -- MAKE-FROM-REAL-IMAG" op)))))
    dispatch)
(define (real-part x)(x 'real-part))
```