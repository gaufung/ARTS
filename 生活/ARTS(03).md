---
date: 2018-10-28
status: public
tags: ARTS
title: ARTS(03)
---
# 1  Algorithm
换零钱问题(`coin exchange`)

>  现在有1元、2元、5元、10元、20元和50元的零钱，现在100元进行零钱转换，那么有多少种找零钱的方法？

这个问题是`SICP`教材上的一道题目，在分析迭代过程和递归过程调用章节中。因此该问题的解法也是采用递归的方式，递归的好处在于很容易描述问题，而迭代往往则是需要一定的逆向思维。但是递归方式要注意递归基，也就是递归的出口。
## 1.1 解题思路
硬币的种类为`1~n`，钱的大小为`s`，对于换零钱`ce(s)`表示`s`元钱可以换成零钱的方法数。任意一种换零钱的方式都可以划分为两部分：
1. 不包含零钱`i`种类的换零钱方式；
2. 将钱`s-d`换为全部种类的硬币的方式，其中`d`为第`i`中零钱的面值。
算法的正确性：将所有换零钱的方式划分为两种：包含零钱`i`和不包含零钱`i`两种。算法的递归基：
- 所剩的钱正好为0， 返回1，表示一种换零钱的方法；
- 所剩的钱小于0或者可选的零钱种类小于0，返回0，表示这不是一种换零钱的方法。

## 实现
```python
def exchange(index):
    if index == 0:
        return 1
   elif index == 1:
       return 2
   elif index == 2:
       return 5
   elif index == 3:
       return 10
   elif index == 4:
       return 20
   else:
       return 50
   
def coin_exchange(amount, index):
    if amount == 0:
        return 1
    if amount<0 or index < 0:
        return 0
    return coin_exchange(amount, index-1) + coin_exchange(amount-exchange(index), index)
```
# 2 Review
## 2.1 [What Programmer Can Do-cache optimization](https://lwn.net/Articles/255364/)
### 2.1.1 L1缓存
我们知道对于多维数组，数据存储是按照一定的循序的，将多维的数据按照一维的内存数组存储，对于一个二维数组，存储的形式如下：
![](./_image/2018-10-30-18-46-15.jpg)
如果访问内存的采用不同的索引顺序，第一种采用行索引优先；另一种采用列索引有限的方式，各自的时间消耗分别为：`0.048s`和`0.127s`，采用行索引优先的方式充分使用了缓存的空间局部性，使缓存行能够保存一行数组。如果数组的大小很大，时间消耗对比将更加突出。
对于矩阵 $A$ 和 $B$  相乘，算法过程大致如下:
```C
for(i = 0; i < N; ++i)
    for(j = 0; j < N; ++j)
        for(k =0; k < N; ++k)
            res[i][j] += mul1[i][k] * mul2[k][j];
```
通过刚刚知道，对于矩阵`mul1`充分使用缓存，但是对于矩阵`mul2`并没有使用缓存；因此需要一种更好的方式访问`mul2`元素：可以使用转置矩阵`mul2`:
$$
(AB)_{ij}=\Sigma_{k=0}^{N-1}a_{ik}b_{jk}^T = a_{i1}b_{j1}^T+a_{i2}b_{j2}^T + \ldots + a_{i(N-1)}b_{j(N-1)}^T
$$
算法过程如下
```C
double tmp[N][N];
for(i =0; i<N; ++i)
    for (j =0; j<N; ++j)
        tmp[i][j] = mul2[j][i]
for(i = 0; i < N; ++i)
    for(j = 0; j < N; ++j)
        for(k =0; k < N; ++k)
            res[i][j] += mul1[i][k] * tmp[j][k];
```
算法的效果是显著地，对比结果如下：
![](./_image/2018-10-30-19-04-21.jpg)
重新拷贝数组仍然不够完美，如果矩阵特别大，将会导致巨大的内存浪费。仔细观察矩阵相乘的数学表达式，可以得到相乘然后相加顺序不影响最后的结果，因此重新调整相加的顺序，使得每个相乘的数据都是在缓存行中。将$A$和$B$矩阵划分若干矩阵块，并且每个矩阵块的行大小为缓存行大小：
![](./_image/2018-10-30-20-20-17.jpg)
在每个子矩阵中， $A_{i*}$和$B_{*j}$都保存在缓存行中，然后用$A_{i*}$中的元素依次乘上$B_{*.j}$一行元素。作为结果$res_{ij}$的一部分。算法流程如下：
```C
#define SM (CLS / sizeof (double))
for (i = 0; i < N; i+=SM)
    for(j=0; j<N; j+=SM)
        for(k=0; k<N; k+= SM)
            for(i2=0, rres = &res[i][j], rmul1 = &mul1[i][k]; i2<SM;
            ++i2; rres += N, rmul1 += N)
                for(k2=0; rmul2 = &mul2[k][j];
                 k2<SM; ++k2, rmul2+=N)
                 for(j2=0; j2<SM; ++j2)
                    rres[j2] += rmul1[k2] * rmul2[j2]
```
### 2.1.2 缓存行空洞
比如下面的结构体
```C
struct foo {
    int a;
    long fill[7];
    int b;
}
```
如果采用64位机器，保证8字节对齐，那么可以知道结构体每个成员在内存中的位置`a:0-4`, `fill: 8-56`和`b: 64-68`,所以总共需要`72`字节，并且存在两个`4`字节的空洞。但是如果我们将`b`移动到`a`附近，那么所有的空洞就不存在了。
### 2.1.3 L1i优化
代码缓存不同于数据缓存，通常代码会被解码成指令，通常要做到以下几点
- 最大可能地简化代码复杂度；
- 代码执行最好是线性，不要出现跳跃；
- 代码对齐也是必要的。
内联函数能够减少代码执行的跳转，但是增加的代码的大小，同时分支预测也是代码优化考虑的范畴， 对于代码
```C
void fct(void){
    ... code block A ...
    if (condition){
        inlfct()
        ... code blcok C ...
    }
}
```
指令优化的结果如下两种：
![](./_image/2018-10-30-22-24-59.jpg)
对于上面的指令顺序，按照代码的原本顺序结果。如果条件很大概率下为`false`，那么下面一种生成的指令则性能更好。
## 2.2 [What programmer can do -multi-threaded optimzation](http://lwn.net/Articles/256433/)
我们都知道，线程是共享全部地址空间，因此多个线程共享共同的变量会带来性能上损失。对于如下如果建议：
1. 将只读变量和读写变量分开(用`const`修饰)，只读变量对于多线程有显著性能优势；
2. 将读写变量组成一个结构体；
3. 将被多个线程使用的读写变量移动到各自的缓存行中；
4. 将独自线程使用的变量移动到线程局部存储`(Thread-Local Storage, TLS)`

# 3 Tips
在中文翻译成英文中，对于较长的句子，使用介词`to`, `with`, `for`等等，将多个长子句合并起来，避免很多的从句，比如下面这一句翻译成英文：
> 下一个月，美国总统唐纳德特朗普将在布宜诺斯艾利斯举行的G-20峰会上，与中国领导人习近平会晤，讨论正在加剧的贸易问题。这则消息公布后，美国股市迎来的短暂的回升。

**如果采用多从句的翻译方式**
> Next month, the U.S. President Donald Trump will meet Chinese leader Xi Jinping at G-20 summit in Buenos Aires and disscuess about intensfiying trade issue. When the new is announced, the U.S. market rebounded briefly. 

将中文翻译成了三句英文，那么如果使用介词，将三句话翻译成一句呢？

**采用介词翻译**
首先确定主旨是关于股市的，所以确定主句是`market`
> The U.S. market rebounded briefly after it was announced that the U.S. President Donald Trump meet with Chinese leader Xi Jinping at next month G-20 summit in Buenuos Aires to dissucss the intensifying trade issue.

# 4 Share
在做选择的时候，常常使用`SWOT`分析法：
- Strength: 擅长的方面
- Weakness: 弱点的方面
- Opportunity: 机会
- Threatness: 威胁
对于不同的选择，使用该方面可以很好的做出更好的权衡。当然也有很简单的方法，在做选择的时候不去关注自己失去的东西，而是看到自己得到的东西。