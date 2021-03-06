---
date: 2018-10-21
status: public
tags: ARTS
title: ARTS(02)
---
# 1 Algorithm
## 1.1 哈希碰撞 
在编程中都会用到哈希函数，它能够将任何对象(整数、字符串、对象实例等等)转换成特定的数值，将无限集合的内容映射到有限的集合中。
![](./_image/2018-10-22-20-27-33.png)
上图为哈希的示意图，既然是无限集合到有限集合的映射，就会带来一个问题，不同的`Key`映射到同一个哈希值，这称为哈希碰撞(`collision`)。
假设每个哈希值出现的概率都是相等的，共有$m$个哈希值，共有$n$个`key`待哈希，那么不出现碰撞的概率$\bar{p}(n)$为
$$
\bar{p}(n) = 1 \times (1-\frac{1}{m}) \times (1 - \frac{2}{m}) \times \ldots \times (1 - \frac{n-1}{m}) = \frac{m!}{m^n(m-n)!}
$$
那么发生碰撞的概率为：$1-\bar{p}(n)= 1 - \frac{m!}{m^n(m-n)!}$
化简上述公式：
由泰勒公式可知
$$e^x=\sum_{k=0}^{+\infty}\frac{x^k}{k!} = 1 + x + \frac{x^2}{2} + \frac{x^3}{6} + \frac{x^4}{24}+\ldots$$
如果$x$非常小，那么$e^x = 1 + x$，通常在哈希中$m$非常大，所以$-\frac{1}{m}$非常小. $e^{-\frac{1}{m}}=1-\frac{1}{m}$；同理$-\frac{2}{m}$ 也是非常小，$e^{-\frac{2}{m}} = 1 -\frac{2}{m}$；依次同理得到$\bar{p}(n)=e^{-\frac{n(n-1)}{2m}}$
所以碰撞概率近似为: $$p(m, n)= 1 - e^{-\frac{n(n-1)}{2m}}$$
## 1.2 MD5
MD5 (Message Digest)算法是一种广泛使用的的算法，该算法将任何输入转换为128bit位的哈希值。共可以有$m=2^{128}$，通过封底估算$m=2^{128}\approx 2^{120} = (2^{10})^{12}\approx (10^3)^{12}=10^{36}$，这个数据什么概念，地球上的细菌细胞数目估计有$5\times 10^{30}$左右，因此该算法的哈希空间非常大。
一般`Unix-like`操作系统提供了`md5sum`命令，输出文件的`md5`值，用来检验该文件是否被修改过。
### 1.2.1 MD5算法流程
`md5`算法将任意长度的消息转换为固定长度输出的128比特位，算法流程如下
- 将输入消息按照`512-bit`块划分（16个32位字)
- 如果输入的消息位数不能`512`位整除，采取如下规则填充：
    - 首先第一个`bit`填充`1`;
    - 再填充若干个`0`，使之消息长度模`512`的值为`448`(512-64);
    - 在用`64`比特位表示原消息长度模$2^{64}$的值。
- 定义四个`Magic Number`, $A, B, C, D$,十六进制表示如下：
    - $A=67452301_{16}$
    - $B=efcdab89_{16}$
    - $C=98badcfe_{16}$
    - $D=10325476_{16}$
- 定义四个函数
    - $F(B, C, D)=(B \wedge C) \vee (\neg B \wedge D) $
    - $G(B, C, D)=(B \wedge D) \vee (C \wedge \neg D)$
    - $H(B, C, D)=B\oplus C \oplus D$
    - $I(B, C, D)=C\oplus (B \wedge \neg D)$
> $\oplus, \wedge, \vee, \neg $分别代表了`XOR`,`AND`,`OR`和`NOT`操作。

- 对512比特位分组的每一组进行相关的迭代操作，最后计算得到的`ABCD`组成`128`比特位为最后MD5值。

### 1.2.2迭代循环伪代码
```
//Note: All variables are unsigned 32 bit and wrap modulo 2^32 when calculating
var int[64] s, K
var int i

//s specifies the per-round shift amounts
s[ 0..15] := { 7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22 }
s[16..31] := { 5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20 }
s[32..47] := { 4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23 }
s[48..63] := { 6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21 }

//Use binary integer part of the sines of integers (Radians) as constants:
for i from 0 to 63
    K[i] := floor(232 × abs(sin(i + 1)))
end for
//(Or just use the following precomputed table):
K[ 0.. 3] := { 0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee }
K[ 4.. 7] := { 0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501 }
K[ 8..11] := { 0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be }
K[12..15] := { 0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821 }
K[16..19] := { 0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa }
K[20..23] := { 0xd62f105d, 0x02441453, 0xd8a1e681, 0xe7d3fbc8 }
K[24..27] := { 0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed }
K[28..31] := { 0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a }
K[32..35] := { 0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c }
K[36..39] := { 0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70 }
K[40..43] := { 0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x04881d05 }
K[44..47] := { 0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665 }
K[48..51] := { 0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039 }
K[52..55] := { 0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1 }
K[56..59] := { 0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1 }
K[60..63] := { 0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391 }

//Initialize variables:
var int a0 := 0x67452301   //A
var int b0 := 0xefcdab89   //B
var int c0 := 0x98badcfe   //C
var int d0 := 0x10325476   //D

//Pre-processing: adding a single 1 bit
append "1" bit to message    
// Notice: the input bytes are considered as bits strings,
//  where the first bit is the most significant bit of the byte.[48]

//Pre-processing: padding with zeros
append "0" bit until message length in bits ≡ 448 (mod 512)
append original length in bits mod 264 to message

//Process the message in successive 512-bit chunks:
for each 512-bit chunk of padded message
    break chunk into sixteen 32-bit words M[j], 0 ≤ j ≤ 15
    //Initialize hash value for this chunk:
    var int A := a0
    var int B := b0
    var int C := c0
    var int D := d0
    //Main loop:
    for i from 0 to 63
        var int F, g
        if 0 ≤ i ≤ 15 then
            F := (B and C) or ((not B) and D)
            g := i
        else if 16 ≤ i ≤ 31 then
            F := (D and B) or ((not D) and C)
            g := (5×i + 1) mod 16
        else if 32 ≤ i ≤ 47 then
            F := B xor C xor D
            g := (3×i + 5) mod 16
        else if 48 ≤ i ≤ 63 then
            F := C xor (B or (not D))
            g := (7×i) mod 16
        //Be wary of the below definitions of a,b,c,d
        F := F + A + K[i] + M[g]
        A := D
        D := C
        C := B
        B := B + leftrotate(F, s[i])
    end for
    //Add this chunk's hash to result so far:
    a0 := a0 + A
    b0 := b0 + B
    c0 := c0 + C
    d0 := d0 + D
end for

var char digest[16] := a0 append b0 append c0 append d0 //(Output is in little-endian)

//leftrotate function definition
leftrotate (x, c)
    return (x << c) binary or (x >> (32-c));
```
    
# 2 Review
**What every programmer should know about memory**
## 2.1 [Virtual Memory](https://lwn.net/Articles/253361/)
上一节知道，由于内存的速度和CPU的数据不匹配，所以使用高速缓存(cache)作为一个中间件，借助缓存的时间局部性(`temporal locality`)和空间局部性(`spatial locality`)，达到了很好的效果。那么如果将抽象层次提升，将内存视为磁盘的缓存，将访问速度慢的内存当速度更慢的磁盘的“高速缓存”，而此时磁盘称为**虚拟内存**。这样做的有三个好处：
1. 采用"缓存"的方式使用内存，可以高效使用内存；
2. 通过虚拟内存，可以获得逻辑上连续的地址空间，而且大小不受物理内存限制；
3. 每个进程拥有各自的虚拟内存，避免的相互干扰。
### 2.1.1 地址翻译
每一个虚拟内存地址都可以缓存到内存中，对于`n`位的操作系统，最大访问的位置$N=2^n-1$，共$2^n$字节；假设物理内存最大位置$M=2^m -1$，共$2^m$字节。虚拟内存使用`页面`(page frame)表示每一个缓存块的大小，大小通常为`4MB`($2^{22}$)或者`4KB`($2^{12}$)大小。
将虚拟内存翻译成物理内存主要工作由`Memory Magagement Unit(MMU)`完成，主要工作如下图所示：
![](./_image/2018-10-24-20-43-56.jpg)
- 虚拟地址低 $p$ 位为虚拟也偏移量，其中 $2^p$ 等于页面大小；
- 虚拟地址高 $n-p$ 位地址为页表(Page Table)的索引；
- 从页表中获取该条目(Entry), 并从中得到物理页面号(Physical Page Number);
- 与虚拟地址中的低 $p$ 位组成物理地址；
- 读取相应的字，返回给CPU。
### 2.1.2 缺页
刚刚讨论的是恰好虚拟内存在物理内存中存在，注意在页表条目中包含了标识符，表明该记录是否有效；如果无效，该虚拟内存页面没有在物理内存中，这时将会引发一个中断，内存中磁盘中读取虚拟页，然后从选择一个物理页作为牺牲页，并更新PTE，然后再一次读取虚拟地址，重复页命中的过程。
### 2.1.3 优化
每一个进程都拥有一个页表，因此保证了每个进程内存空间不相互影响；页表是存储在内存中的，但是每次读取PTE，还是消耗更长的周期，所以可以选择部分PTE存储在`L1`缓存中，称为翻译后备缓冲器(`Translation Lookaside Buffer, TLB`)。
通常页大小为4KB($2^{12}$)，对于`32`位操作系统，那么页表需要准备$2^{20}$条目，这现实中不可行，因此现在操作系统选择多级页表：
![](./_image/2018-10-25-15-37-53.jpg)
虚拟地址被划分为五个部分，其中`Level1`到`Level4`是各个层级的页表，地址翻译从`Level4`开始，每一个条目指向下一个层级的页表。`Level1`条目指向的是物理页编号`PPN`，一旦出现页表条目标记无效，则表明该虚拟内存没有缓存到内存中。
通常高速缓存(Cache)保存是字节的物理地址，而不是虚拟地址。在地址翻译获得物理地址后，先从高速缓存中获取字，缓存不命中从内存中读取。
## 2.2 [NUMA](https://lwn.net/Articles/254445/)
非统一内存访问(`Non Uniform Memory Access`)是用于多处理器内存设计架构，之前知道，访问不同位置的物理内存的消耗取决于访问的位置。从性能上考虑，现代处理器设计都拥有本地内存，这些比访问其他处理器的局部内存消耗低得多。
多处理器重要的一点就是如何设计节点的拓扑结构，最有效的方式采用`hypercube`，如下图所示：
![](./_image/2018-10-26-20-11-30.jpg)
对于拥有$2^C$节点的拓扑结构，其中$C$为每一个节点连接其他节点的最大连接数。
既然硬件支持`NUMA`架构，那么操作系统级别也就要支持该架构，如果某个进程给定了处理器，那么进程的物理地址空间就来自局部内存；操作系统不应该将一个进程或者线程从一个处理转向另一个处理器。
在` /sys/devices/system/cpu/cpu*/cache`下可以查看处理器的缓存的信息：
![](./_image/2018-10-26-21-21-40.jpg)
- 这是一个四核的CPU；
- 每一个核有三个cache:`L1i`，`L1d`和`L2`;
- `L1i`和`L1d`缓存是独享的，因为每一个比特位都是1；
- `cpu0`和`cpu1`共享`L2`缓存，而`cpu2`和`cpu3`共享`L3`缓存。
# 3 Tips
有时候在使用某些命令的时候，由于权限不够，需要以`sudo`的形式执行。但是以`sudo`执行命令的时候却出现了`command not found`。这是因为系统由于安全的考虑，`sudo`命令将会在一个最小化的环境中执行。其中`/etc/sudoers`文件中的`secure_path`说明`sudo`可执行命令的路径。
```shell
# ....
Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
#....
```
将要执行的命令添加到`secure_path`中即可使用`sudo`执行该命令。
# 4 Share
在与他人交流问题的时候，首先要**给出结论**，然后再依次给出理由，分点来讲。