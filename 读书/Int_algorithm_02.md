---
date: 2016-09-26 20:45
status: public
tags: 学习
title: MIT算法导论课程学习2
---

# Chapter 08  
1. Inorder-tree-walk takes $T(n)$ Time 
proof: *substitute :*  $T(n)=T(k)+T(n-k-1)+d$ Supposed that $T(n)=(c+d)n+c$ 
$$T(n)=T(k)+T(n-k-1)+d=((c+d)k+c)+((c+d)(n-k-1)+c)+d = (c+d)n+c-(c+d)+c+d=(c+d)n+c$$ 

2. In a binary search tree: Operations likes: SEARCH, MINIMUM, MAXIMUM, SUCCESSSOR and PREDECESSPR cost times are $T(h)$, whree h is the height of this bst. 

3. Insert operation
```
y <- nil
x <- root[T]
while x != nil
    do y <-x 
        if key[z] < key[x]
            then x <- left[x]
            else x <= right[x]
p[z] <- y
if y=nil
    then root[T] <- z
    else if key[z] < key[y]
        then left[y] <- z
        else right[y] <- z
```  
4. Delete operation
Search the pointer 
    + if pointer has no children: delete the node safely.
    + if pointer has only one child : swap the value between pointer and its child.
    + if pointer has two children: find the node's successor. swap them. then delete the successor .

5. Height of randomly built binary search tree  
**Jensen's inequality:** $f[E(X)] \le E[F(x)]$  for any convex function $f$ and random variable $x$. Random variable $Y_n=2^{X_n}$ where $X_n$is the random variable denoting the height of the BST.  
**Lemma**  
$$f[E(x)]=f(\sum_{k=-\infty}^{\infty}k \cdot Pr \lbrace X=k\rbrace) \le \sum_{k=-\infty}^{\infty}f(x) \times Pr \lbrace X=k \rbrace = E[f(x)]$$  
let $Y_n=2^{X_n}$. $X_n=1+max\lbrace X_{k-1},X_{n-k} \rbrace$ So
$$Y_n=2^{x_n}=2^{1+max\lbrace X_{k-1},X_{n-k} \rbrace}=2 \times max \lbrace 2^{X_{k-1}}, 2^{X_{n-k}} \rbrace = 2 \cdot max \lbrace Y_{k-1},Y_{n-k} \rbrace$$   
let $Z_{nk}=1$ if the root has rank $k$, otherwise $Z_{nk}=0$ $E[Z_{nk}]=\frac{1}{n}$ 
So.$$Y_n=\sum_{k=1}^{n}Z_{nk}(2\cdot max \lbrace Y_{k-1},Y_{n-k} \rbrace)$$  
the exceptation of $Y_{n}$
$$E[Y_{n}]=E\lbrack \sum_{k=1}^{n}Z_{nk}(2 \cdot max \lbrace Y_{k-1},Y_{n-k}\rbrace )\rbrack$$ 
$$=2\sum_{k=1}^{n}E\lbrack Z_{nk}(max\lbrace Y_{k-1},Y_{n-k} \rbrace ) \rbrack$$
$$\le \frac{2}{n}\sum_{k=1}^{n}E\lbrack Y_{k-1},Y_{n-k} \rbrack =\frac{4}{n}\\sum_{k=0}^{n-1}E[Y_k]$$ 
*substitute :* $E[Y_n]\le cn^{3}$   
$$E[Y_n]=\frac{4}{n}\sum_{k=0}^{n-1}E[Y_k] \le \frac{4}{n}\sum_{k=0}^{n-1}ck^3 \le \frac{4c}{n}\int_{0}^{n}x^3dx=cn^3$$
Conclusion：
$$2^{E[X_n]} \le cn^3 \Rightarrow E[X_n] \le 3lgn$$   

# Chapter 09
1. Red-Black tree properties
    + Every node is either red or black
    + The root and leaves(NIL's) are black
    + if a node is red, then its parent is black
    + all simple paths from any node x to a descendant leaf have the same number of black nodes =$black-height(x)$  

2. Height of a red-black tree
a red-black tree with $n$ keys has height $$h \le 2lg(n+1)$$  
merge red nodes into their black:
![](</_image/algorthm/5.png>)
![](</_image/algorthm/6.png>) 
a tree in which each node has 2,3,or 4 children. and has uniform depth $h'$ of leaves. So we had know that $h' \ge h/2$. Each tree has $n+1$ leaves So:
$$n+1 \ge 2^{h'} \Rightarrow lg(n+1) \ge h' \ge h/2 \Rightarrow h \le 2lg(n+1)$$  

3. Rotations 
![](</_image/algorthm/7.png>)  

# Chapter 10 
Dynamic order statistics 
1.  OS-Select(i,S): return the $i$th smallest element in the dynamic set $S$
    OS-Rank(x,S)：return the rank of $x \in S$ in the sorted order of S's elements   

2. **Solution**: Use a red-black tree for the set S, but keep subtree sizes in the nodes  
![](</_image/algorthm/8.png>)    

$size[x]=size[left[x]]+size[right[x]]+1$  
3. **Algorithm**  
```
OS-Select(i,x) ==> i th smallest element in the subtree rooted at x 
k <- size[left[x]] +1 
if i = k then return x
if i < k 
    return return OS-Select(left[x],i)
    else return Os-Select(right[x],i-k)
```    
4. Data-structure augmentation 
Methodology 
    + choose an underlying data structure (red-black trees)
    + determine *additional infomation* to be stroed in the data structure
    + Verify that this infomation can be mationtained for modifying operation
    + Develop new dynamic-set operations that use this information 
5. Interval tree  

**addtional information** stroe in each node x the largest value $m[x]$ in the substree root at x, as well as the interval $int[x]$ correspoinding to the key.  
![](</_image/algorthm/9.png>)   

**Algorithm**
```
Interval-search(i)
x <- root
while x != nil and (low[i] > high [int[x]] or low[int[x]] > high [i] )
    do ==> i and int[x] don't overlap
        if left[x] != nil and low[i] <= m[left[x]]
            then x <- left[x]
            else x <- right[x]
return x
```

**Correctness**
+ if the search goes right, then $\lbrace i' \in L : i' overlaps i\rbrace = \oslash $ 
+ if the search goes left, then if $\lbrace i' \in L: i' overlaps i\rbrace= \oslash \Rightarrow \lbrace i' \in R: i' overlaps i\rbrace=\oslash$