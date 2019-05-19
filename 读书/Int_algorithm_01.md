---
date: 2016-09-10 07:54
status: public
tags: 读书
title: MIT算法导论课程学习1
---

这是一门在网易公开课上的开放的课程，[地址](http://open.163.com/special/opencourse/algorithms.html) ，已经在手机上缓冲全部课程，争取每看完一章，做笔记总结。
# Chapter 01
1. Insert Sort
Pseudocode:
``` pseudocode
INSERTION-SORT (A,n)  ===> A[1,n]
for j <- 2 to n
    do key <- A[j]
        i <- j-1
        while i > 0 and A[i] > key
            do A[j+1] <- A[j]
                i <- i-1
        A[i+1]=key
```
2. Kinds of analysis
    + worst-case: usually
    + average-case: excepted time of algorithm,statiscal distrubution of inputs
    + best-case:bogus     

3. $\Theta$-notation  
$\Theta(g(n))=f(n)$: drop low-order terms;ingore leading constant 

4. Merge sort 
Merge-Sort A[1...n]
    1 if n=1,done.
    2 Recursively sort A[1..$\lceil n/2 \rceil$] and A[$\lceil n/2 \rceil$ +1 .. n]
    3 "Merge" the 2 sorted lists.
5. Merge sort analysis 
$T(n)=2T(\frac{n}{2})+\Theta(n)$  
    + Recursion tree  
    ![](</_image/algorthm/1.png>)

# Chapter 02 
1. Master method
Applys to recurrences of the form $$T(n)=aT(b/n)+f(n)$$
    + $f(n)=O(n^{log_ba-\epsilon})$  
    for some constant $\epsilon > 0$. $f(n)$ glows polynomially slower than $n^{log_ba-\epsilon}$
    **Solution**: $T(n)=\Theta(n^{log_ba-\epsilon})$
    + $f(n)=\Theta(n^{log_ba}(\lg{n})^{k})$  
    for some constant $k \ge 0$. $f(n)$ and $n^{log_ba-\epsilon}$ grow at similar rates
    **Solution**: $T(n)=\Theta(n^{log_ba} (\lg{n})^{k+1})$
    +  $f(n)=\Omega(n^{log_ba+\epsilon})$  
    for some constant $\epsilon >0 $. $f(n)$ grows polynomially faster than $n^{log_ba-\epsilon}$
    and $f(n)$ satisfies the **regularity condition** that $af(n/b) \le cf(n)$ for some constant c <1 
    **Solution**: $T(n) = \Theta(f(n))$   
2. General method (Akra - Bazzi)
    $$T(n)=\sum_{i=1}^{k}a_{i}T(\frac{n}{b_{i}})+f(n)$$
    let $p$ be the unique solution to $$\sum_{i=1}^{k}(\frac{a_{i}}{b_{i}^{p}})=1$$.
    Use $n_{p}$ instead of $n^{log_ba}$ in master method.
3. Theorem   
![](</_image/algorthm/2.png>)   

# Chapter 03
Divide and Conquer
        + *Divide* the problem into subproblems
        + *Conquer* the subproblems by solving them recursively
        + *Combine* subprolems' solutions
1. Binary search  
$T(n)=1T(n/2)+\Theta(1)$   
$n^{log_ba}=n^{log_21}=n^{0}=1 \Rightarrow Case2 (k=0) \Rightarrow T(n)=\Theta(lgn)$  
2. Powering a number
Compute $a^{n}$, where $b \in \mathbb{N}$ 
    + naive algorithm: $\Theta(n)$
    + Divide and conquer algorhtm:
    $$a^{n}=\begin{equation}\begin{cases} a^{n/2}\times a^{n/2}  \\\ a^{(n-1)/2} \times a^{(n-1)/2} \times a \end{cases} \end{equation}$$ 
    $T(n)=T(n/2)+\Theta(1)$  
3. Fibonacci numbers 
$F_{n+1}=F_{n}+F{n-1}$, $F_{0}=0,F_{1}=1$ 
    + naive recursinve algorithm: $\Omega(\phi^{n})$, where $\phi=(1+\sqrt{5})/2$ is golden ratio.
    + iteration:   
    
    ```python
    def fibonacci(n):
        if(n==0):
            return 0
        h=0
        g=1
        i=0
        while(i<n):
            g=g+h
            h=g-h
            i=i+1
        return g    
    ```
    $T(n)=\Theta(n)$
    + Divide-and-conquer
    $$\begin{vmatrix} F_{n+1} \\\ F_{n} \end{vmatrix} = \begin{vmatrix} 1 \\ 1 \\\ 1 \\ 0 \end{vmatrix} \times \begin{vmatrix} F_{n} \\\ F_{n-1} \end{vmatrix} = \begin{vmatrix} 1 \\ 1 \\\ 1 \\ 0 \end{vmatrix} ^{n} \times \begin{vmatrix} F_{1} \\\ F_{0} \end{vmatrix}$$ like powering a number instead of a matrix. $T(n)=\log n$  

4. Matrix Multiplication 
Input: $A=[a_{ij}],B=[b_{ij}]$   
Output:$C=[c_{ij}]$ 
$$c_{ij}=\sum_{k=1}^{n}a_{ik} \times b_{kj}$$   
    + standard algorithm
    $T(n)=\Theta(n^{3})$ 
    + Divide-and-conquer algorithm
    ![](</_image/algorthm/3.png>) 
    $$\begin{equation} \begin{cases} r=ae+bg \\\ s=af+bh \\\ t=ce+dh \\\ u=cf+dg \end{cases} \end{equation}$$ 
    8 multis of $(n/2) \times (n/2)$ submatrices. 4 adds of $(n/2) \times (n/2)$ submatrices. 
    $$T(n)=8T(n/2)+\Theta(n^{2})$$ 
    $n^{log_ba}=n^{log_28}=n^3 \Rightarrow Case1 \Rightarrow T(n)=\Theta(n^3)$
    + Strassen's idea
    $$\begin{equation}\begin{cases} P_{1}=a(f-g) \\\ P_{2}=(a+b)h \\\ P_{3}=(c+d)e \\\ P_{4}=d(g-e) \\\  P_{5}=(a+d)(e+h) \\\ P_{6}=(b-d)(g+h) \\\ P_{7}=(a-c)(e+f) \end{cases}\end{equation}$$
    $$\begin{equation} \begin{cases} r=P_{5}+P_{4}-P_{2}+P_{6} \\\ s=P_{1}+P_{2} \\\ t = P_{3}+P_{4} \\\  u = P_{5}+P_{1}-P_{3}-P_{7} \end{cases} \end{equation}$$
    $$T(n)=7T(n/2)+\Theta(n^2)$$,
    $n^{log_ba}=n^{log_27} \sim n^{2.81} \Rightarrow Case 1 \Rightarrow n^{lg7}$
    
# Chapter 04  
Quick Sort 
1. Divide and Conquer
    + *Divide*:Partition the array into two subarrays around a **pivot** x 
    + *Conquer*：Recursively sort the two subarray
    + *Combine*： Trivial
2. Algorithm 
```
Partition(A,p,q) ===> A[p,q]
    x <- A[p]    ===> pivot=A[p]
    i <- p
    for j <- p+1 to q
        do if A[j] <= x
            then i <- i+1
                exchange A[i] <-> A[j]
    echange A[p] <-> A[i]
    return i
```
**Invariant**:  A[p+1，i] <=pivot , A[i+1,j] >=pivot 

**montonicity**: q-j 

3. Time cost analysis 
    + Worst-case 
    $T(n)=T(0)+T(n-1)+\Theta(n) = \Theta(n^2)$ 
    + Best-case 
    $T(n)=2(T/n)+\Theta(n)=\Theta(nlogn)$  
4. Randomized quicksort
5. T(n) 
<a href="http://gaufung.info/post/shu-ju-jie-gou-shi-yi-1#toc_1">Time cost </a>  

# Chapter 05 
1. The best worst-case running time of comparison sorting. 
Proof: There are $n!$ possible permutations for input size $n$. Each comparison divide all permutations into 2 parts and each part contains same counts of permutations. The time cost is the times of the dividsion, so $T(n)=log_2(n!)$. According to Stirling's formula $ n! \ge (n/e)^n $. So $T(n) \ge log_2((n/e)^n)=nlogn-nloge=\Theta(nlogn) $.  

2. Counting Sort:
Stable sort 
input: A[1...n],where $A[j] \in {1,2..k}$
output: B[1..n],sorted.
auxiliary storage: C[1..k]  
```pseudocode
for i<-1 to k
    do C[i]<-0
for j <- 1 to n
    do C[A[j]] <- C[A[j]]+1
for i<-2 to k
    do C[i] <- C[i]+C[i-1]
for j<-n downto 1
    do B[C[A[j]]] <- A[j]
        C[A[j]] <- C[A[j]]-1
```
Analysis : Time cost$T(n)=\Theta(n+k)$, if $k=\Theta(n)$, then countiung sort takes $\Theta(n)$ time.

3. Radix sort
Digit-by-digit sort. 
Good idea:Sort on least-significant digit first with auxiliary stable sort. 
![](</_image/algorthm/4.png)  

# Chapter 06 
1. Order statistic
select $i$th smallest of n elements 
**Naive algorithm** Sort and index $i$th elment. worst-case running time $T(n)=\Theta(nlogn)+\Theta(1)=\Theta(nlogn)$  
## randomized divide-and-conquer algortihm 
like quick-sort, when partition the array, we know the rank of the pivot in this array. 
    + If pivot's rank $\gt$ i. recurse the first part to find the i rank element.   
    + If pivot's rank $\lt$ i. recurse the second part to find the i-k rank elemenet.   
    + If pivot's rank $=$ i. just return the pivot.   
### summary of randomized order-statisc selection
    + work fast: linear **excepted**  time
    + Excellent algorithm in practice.
    + wrost case is **very bad** $\Theta(n^2)$   
## worst-case linear-time order statistic
1. divide the n elements into group 5, find the median of each 5-element group by rote
2. recursively selective teh median x of the $\lfloor n/5 \rfloor$ group median to be the pivot
3. partition around the pivot x. Let $k=rank(k)$ 
    + if i=k then return x
    + else if i<k
        then recursively select the ith small element in the lower part. 
        else recursively select the (i-k)th smallest element in the upper part.
### analysis teh liner-time order statistic  
each recursion, 1/4 part of data was dropped.
$$T(n)=T(\frac{1}{5}n)+T(\frac{3}{4}n)+\Theta(n)$$ 
Substitution:
$T(n)\le cn$: SO $T(n)\le \frac{1}{5}n +\frac{3}{4}cn+\Theta(n) = \frac{19}{20}cn+\Theta(n)=cn-(\frac{1}{20}cn-\Theta(n)) \le cn$ if c is chosen large enough to handle both the $\Theta(n) $ and the initial conditions    

# Chapter 07 
## Hash function  

Use a hash function $h$ to map the universe $U$ of all keys into ${0,1,...,m-1}$. collision occurs. 
1. Resloving collsion by chaining: record in the same slot are linked into a list.  
2. **load factor** 
$$\alpha=\frac{n}{m}$$ where $n$ the number of keys in the table, $m$ the number of slots. Search cost $\Theta(1+\alpha)$  
## Hashing function choice
1. Division method
    $$h(k)=k mod m$$
> if $m=2^r$, then the hash doesn't even depend on all the bits of k.   
pick m to be a **prime** not too close to a power of 2 or 10.
2. Multiplication method
$m=2^r$, our computer has $w$-bit words.
$$h(k)=(A \times k mod 2^w) rsh (w-r)$$
where rsh is the "bit-wise right-shift" operator and A is an odd integer in the range $2^{w-1} \lt A \lt 2^w$   

## collision 
1. open addressing 
no storage is used outside of the hash table itself. 
Insertion systematically probes the table until an empty slot is found.
The probe sequence $[h(k,0),h(k,1),...,h(k,m-1)]$   
2. Probing strategies 
### Linear probing 
$h(k,i)=(h(k)^{'}+i)$  mod $m$
### double hashing 
$h(k,i)=(h_1(k)+i \times h_2(k))$ mod $m$   
3. analysis of open addressing 
if load factor $\alpha = \frac{n}{m} \lt 1$ the excepted number of probes in an unsuccessfuk search is at most $\frac{1}{1-\alpha}$ 
if the table is half full. then teh excepted number of probes is 1/(1-0.5)=2