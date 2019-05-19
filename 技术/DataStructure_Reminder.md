---
date: 2016-08-29 21:27
status: public
tags: Algorithms
title: 数据结构拾遗(0)
---

最近忙着9月份的校招，所以最近一直在复习数据结构相关知识。其实从2013年开始就断断续续的学习清华大学MOOC[数据结构](https://www.xuetangx.com/courses/course-v1:TsinghuaX+30240184X+sp/about)这门课程。
课程的内容是用C++写的，也花了一段时间改写成C#，已经放在github上[Data Structure](https://github.com/gaufung/DS) 最近也做了一些复习，算作拾遗。
# 0 Time cost
> 一个合格的程序猿基本的一点是能够明确自己所写的算法的时间复杂度。 

+ 常数$O(1)$
ordinaryElement 从集合中筛选非最大和非最小元素
```C#
ordinaryElement(S[],n)
any three elements x,y and z belong to S, 
sorting them a,b,c
output b
```
+ $O(log(n))$
非负整数，统计二进制位数
```C++
int countOnes(unsigned int n){
    int ones=0;
    while(0<n){
        ones +=(1&n);
        n>>=1;
    }
    return ones;
}
```
+ $O(n)$
+ Polynomial(n)  
算法度量为$T(n)=O(f(n))$，而且$f(x)$为多项式，一般认为是tractable
+ $2^{n}$  
一般认为是intractable  

# 1 Recursion
递归算法的递归调用恰好在最后一步调用，则成为尾递归（tail recursion)。 一般来讲，尾递归都能转换为迭代的形式（借助栈结构）。

# 2 Back-of-envelope Calculation
+ $1 day = 24 h \times 60min \times 60sec \simeq 10^{5}sec$ 
+ $1 life = 1 century = 100 yr \times 365 \simeq 3 \times 10^{9}sec $
+ $ 300 yr \simeq 10^{10} sec \simeq (1  googel)^{1/10}sec$  


# 3 Binary Search
二分查找一般算法
```C#
public int BinarySearch(int[] array,int lo,int hi,int value){
    while(lo<hi){
        int mi=(lo+hi)>>1;
        if(arrray[mi]==value){
            return mi;
        }else(array[mi]>value){
            hi=mi;
        }else{
            lo=mi+1;
        }
    }
    return -1;
}
```
但是相对而言，有两个缺点
1. 查找失败，返回-1，那能否返回查找失败的位置
2. 查找成功，那么如果有多个，能否找到秩最大的那个
```C#
public int BinarySearch(int[] array,int lo,int hi,int value){
    while(lo<hi){
        int mi=(lo+hi)>>1;
        if(value<array[mi]){
            hi=mi;
        }else{
            lo=mi+1;
        }
    }
    return --lo;
}
```
算法不变性：对于[0,lo)<=value,而对于[hi,n)>value。在算法迭代过程中，不变性保持，而lo与hi之间距离逐渐减少，直至为0，**--lo**则为定义的秩。  

# 4 Merge Sort
单向链表的归并排序。一般来讲，归并排序采用的是数组的表达形式，应为可以很高效的循秩访问（call by rank),但是单向链表却往往缺乏该优势，好在归并排序在Merge过程中，往往是从两个排序好的顺序表从头像尾部扫描，所以单向链表的归并排序也得以完成。但是要注意的一点是如何判断其中的某一个序列已经扫描完毕，所以在Divide过程中，需要将拆分的两个子串的尾部置Null。
```C#
private ListNode MergeSort(ListNode node){
        ListNode slow=node;
        ListNode fast=node.next;
        //if only one node
        if(fast==null)
            return node;
        while(fast!=null){
            fast=fast.next;
            if(fast!=null){
                fast=fast.next;
                slow=slow.next;
            }
        }
        ListNode leftPointer=node;
        ListNode rightPointer=slow.next;
        //the key point 
        slow.next=null;
        ListNode left=MergeSort(leftPointer);
        ListNode right=MergeSort(rightPointer);
        return Merge(left,right);
    }
    private ListNode Merge(ListNode left,ListNode right){
        if(left==null)
            return right;
        if(right==null)
            return left;
        //ensure the left branch is smaller
        if(left.val>right.val){
            return Merge(right,left);
        }
        ListNode backup=left;
        ListNode p=left;
        left=left.next;
        while(left!=null && right!=null){
            if(left.val <= right.val){
                left=left.next;
                p=p.next;
            }else{
                ListNode temp=right;
                right=right.next;
                temp.next=left;
                p.next=temp;
                p=p.next;
            }
        }
        if(left==null){
            p.next=right;
        }
        return backup;    
    }
```

# 5  Stack
栈操作的相关场景主要有以下几个
1. 逆序输出：虽然有明确的算法，但其解答却以线性序列的形式给出，其次，该序列都是依逆序计算输出。
2. 递归嵌套：具有相似性的问题多可嵌套的递归描述，但因分支位置和嵌套并不固定，其递归算法的复杂度不易控制。
    + stack permutation 
    + 括号匹配
3. 延迟缓冲：输入可分解为多个单元并通过迭代一次扫描处理，但过程中各部的处理计算往往滞后于扫描的进度，需要待到必要的信息才能实施计算
    + 表达式求值
    基本思路是将运算符和运算数分别使用两个栈存放，并且事先确定好各个运算符的优先级，通过扫描整个表达式，借助栈工具，完成表达式求值
    运算符比较表：
    
    ![](</_image/dsa/1.png>)
    
    当栈顶运算符大于扫描时当前运算符，从弹出栈顶操作符，并且从操作数栈中Pop出相应数量的操作数，计算结果并入操作数栈；如果栈顶运算符小于当前运算符，运算符入栈，继续扫描表达式。
    从上表中可以看出，当左括号"("为栈顶操作符时，所有操作符均优先级均大于，并入栈。而当左括号为当前操作符时，均入栈。如果该表达式合法，那么如果当前操作符为右括号时，其栈顶操作符必为左括号，处理步骤只需将这一对括号弹出。
    该处理步骤也同时生成Reverse Polish Notation(逆波兰表达式,后缀表达式)

```C#
 public static void Caluc(string expression)
{
            StringBuilder rpn = new StringBuilder();
            Stack<int> opnd = StackVectorImpl<int>.StackFactory();
            Stack<Char> optr = StackVectorImpl<Char>.StackFactory();
            optr.Push('\0');
            int counter = 0;
            while (!optr.Empty)
            {
                // if(counter>expression.Length) break;
                if (counter < expression.Length && Isdigit(expression[counter]))
                {
                    ReadNumber(ref counter, expression, opnd);
                    rpn.Append(opnd.Top);
                }
                else
                {
                    char compareOp = counter == expression.Length ? '\0' : expression[counter];
                    switch (OrderBetween(optr.Top, compareOp))
                    {
                        case '<':
                            optr.Push(expression[counter]);
                            counter++;
                            break;
                        case '=':
                            optr.Pop();
                            counter++;
                            break;
                        case '>':
                            char op = optr.Pop();
                            rpn.Append(op);
                            if ('!' == op)
                            {
                                int pOnd = opnd.Pop();
                                opnd.Push(Caluc(pOnd));
                            }
                            else
                            {
                                int pOnd2 = opnd.Pop();
                                int pOnd1 = opnd.Pop();
                                opnd.Push(Caluc(pOnd1, op, pOnd2));
                            }
                            break;
                    }
                }
            }
            Reuslt = opnd.Pop();
            Rpn = rpn.ToString();
        }
        
```

# 6 Quick Sort  
快排是为数不多的算法时间复杂度为$nlog(n)$的排序算法，其中于归并算法类似，主要是通过Divide and Conquer 的方式完成高效的排序算法。然后与归并排序的不同点在于，快排的主要工作量是在于高效的寻找到轴点pivot的位置i,那么[lo,i)<=pivot 而(i,hi]>=pivot。 那么pivot的位置确定，通多递归调用quickSort对左右两个区间进行排序，完成排序算法。
```C#
public void QuickSort(int[] nums,int lo,int hi){
    if(lo>=hi){
        //recursion base
        return;
    }else{
        // call the parition function
        int mi=Parition(nums,lo,hi);
        QuickSort(nums,lo,mi-1);
        QuickSort(nums,mi+1,hi);
    }
}
```
对于Parition算法而言，从区间的两端向中间搜索，在搜索的过程中的不变性是[lo,i)<pivot && (j,hi]>pivot,单调性为区间[i,j]逐渐变小，直至为1，算法停止。i=j的位置则为pivot的位置。

```C#
private int Parition(int[] nums,int lo,int hi)
{
    //choose the first element as the pivot
    int pivot=nums[lo];
    int i=lo;
    int j=hi;
    int temp=lo;
    while(i<j)
    {
        while(nums[j]>=pivot)
        {
            j--;
        }
        nums[temp]=nums[j];
        temp=j;
        while(nums[lo]<=pivot){
            i++;
        }
        nums[temp]=nums[i];
        temp=i;
    }
    nums[i]=pivot;
    return i;
}
```
时间复杂度分析：从平均意义上而言
$$T(n)=(n+1)+\frac{1}{n} \times \sum_{k=0}^{n-1}[T(k)+T(n-k-1)]$$
对于$k$而言，求和部分是相等的
$$T(n)=(n+1)+\frac{2}{n} \times \sum_{k=0}^{n-1}T(k)$$
等式两边同时乘以$n$
$$n \times T(n)=n \times (n+1) + 2 \times \sum_{k=0}^{n-1}T(k)$$
对于n-1而言
$$(n-1) \times T(n-1) = (n-1) * n + 2 \times \sum_{k=0}^{n-2}T(k)$$
两式相减
$$n \times T(n) - (n-1) \times T(n-1) = 2 \times n + 2 \times T(n-1)$$
整理得
$$n \times T(n) - (n+1) \times T(n-1) = 2 *n$$
等号两边同时除以$n(n+1)$ 得
$$\frac{T(n)}{(n+1)}= \frac{2}{n+1} + \frac{T(n-1)}{n} = \frac{2}{n+1}+\frac{2}{n}+\frac{T(n-2)}{n-1}=....$$
所以
$$\frac{T(n)}{n+1}=\frac{2}{n+1}+\frac{2}{n}+..+\frac{T(0)}{1}=(2 \times ln2) \times logn=1.39logn$$
得到：$T(n)=1.39(n+1)logn$