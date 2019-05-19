---
date: 2016-10-07 20:32
status: public
tags: 学习
title: MIT算法导论课程学习4
---

# Chapter12
Amortized analysis
1. Dynamic table 
We increase the table size by 2 times when the table is full 
    + The initial table size is 1
    + Insert the value when the table has free slot which takes time:  $O(1)$
    + Insert the value when the table has been full
        + Increase the table size by 2 times
        + Copy original elements from old table to new table 
        + Insert value into new table.
    + Let $c_i$ represents the $i-th$ insertion:
        + $c_i=i$ if $i-1$ is an exact power of 2.
        + 1 otherwise
    + Total time cost：$$\sum_{i=1}^{n}c_i \le n+\sum_{j=0}^{\lfloor lg(n-1) \rfloor}2^j \le n+2^{\lfloor lg(n-1) \rfloor} \le 3n = \Theta(n)$$ 
    + the average cost of each dynamic-table operation is $$\Theta(n)/n=\Theta(1)$$  
2. amortization arguments 
    + *aggerate* method
    + *accounting* method
    + *potential* method