---
date: 2016-10-03 20:23
status: public
tags: 学习
title: MIT算法导论课程学习3
---

# Chapter11 
1. 衡量一个数论算法所要求的“位操作”比较适宜
2. 基础初等数论概念
    + $b | a$ (d整除a) $\Rightarrow a=k \cdot d, k$为整数, d是a的约数
    + 除法定理：$\forall a,\forall n, \exists q,r \Rightarrow a=qn+r \Rightarrow q=\lfloor a/n \rfloor $除法的商，$r=amod n$除法的余数
    + 整数的划分：$[a]_n=\lbrace a+kn;k \in Z \rbrace \Rightarrow Z_n=\lbrace [a]_n;0 \le a \le n-1 \rbrace$ 
3. 公约数和最大公约数
    + 对任意$x,y$有：$d|a and d|b \Rightarrow d|(ax+by) $
    + $a|b ,and , b|a \Rightarrow a=\pm b$ 
    + gcd(a,b)
    $$\begin{equation} \begin{cases} gcd(a,b)=gcd(b,a) \\\ gcd(a,b)=gcd(-a,b) \\\ gcd(a,b)=gcd(|a|,|b|) \\\ gcd(a,0)=|a| \\\ gcd(a,ka)=a \end{cases} \end{equation}$$ 
    + $a,b$都不是为0的任意正整数，gcd(a,b)是a,b的线性组合$\lbrace ax+by;x,y \in Z \rbrace$的最小值 
    + $$\begin{equation} \begin{cases} d|a,and,d|b \Rightarrow d|gcd(a,b) \\\ gcd(na,nb) \Rightarrow ngcd(a,b) \\\ n|ab,and,gcd(a,n)=1 \Rightarrow n|b \end{cases} \end{equation}$$ 
    
+ 互质数：gcd(a,b)=1,则a,b互质。 
+ 唯一因子分解：$\forall p $素数，合数$a,\Rightarrow a=p_{1}^{e_1}\cdot p_{2}^{e_2}\cdot\cdot\cdot p_{n}^{e_n}$ 
4. 证明素数有无穷多个
Proof：反证法
假设素数有限多个：A：2,3,5,...,p
令$S=2\times3\times5\times \cdot\cdot\cdot \times p+1$,如果S为素数，S>p,矛盾。如果S为合数，S不能被（2,3,5,...,p）中任何一个整除，必存在一个素数整除S，矛盾。 
5. 最大公约数gcd(a,b)
$$\begin{equation} \begin{cases} a=p_{1}^{e_1}p_{2}^{e_2}\cdot\cdot p_r^{e_r} \\\ b=p_1^{f_1}p_2^{f_2} \cdot \cdot p_r^{e_r} \\\ \Rightarrow gcd(a,b)=p_1^{min(e_1,f_1)} p_2^{min(e_2,f_2)} \cdot \cdot p_r^{min(e_r,f_r)}\end{cases}\end{equation}$$ 
递归定理：gcd(a,b)=gcd(b,a mod b)
```
Euclid(a,b)
if b=0
    reutrn a
esle return Euclid(b,a mod b)
```