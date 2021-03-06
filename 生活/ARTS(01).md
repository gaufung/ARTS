---
date: 2018-10-14
status: public
tags: ARTS
title: ARTS(01)
---
# 1 Algorithm
*RSA非对称加密算法*
## 欧拉函数
对于任意整数`n`，小于`n`且与`n`互质数的数量称为: $\varphi(n)$， 有以下几个推论
1. 对于质数`n`:  $\varphi(n) = n-1 = n(1-\frac{1}{n})$
2. 如果`n`可以分为两个互质的数积：$n=p_1\times p_2 \rightarrow \varphi(n)=\varphi(p_1)\times \varphi(p_2)$
3. 所有的数都可以分解为任意数量的质数的积: $n=p_1p_2\ldots p_k \rightarrow \varphi(n)=p_1(1-\frac{1}{p_1})p_2(1-\frac{1}{p_2}) \ldots p_k(1-\frac{1}{p_k})=p_1p_2\ldots p_k(1-\frac{1}{p_1})\ldots (1-\frac{1}{p_k}) = n(1-\frac{1}{p_1})\ldots (1-\frac{1}{p_k}) $

## 欧拉定理
> $n, a$为正整数，并且$n, a$互质，则$a^{\varphi(n)} = 1 (mod \space n)$

## 模反元素
> 如果两个正整数$a$和$n$互质，那么一定可以找到整数$b$，使得 $ab-1$ 被$n$整除，或者说$ab$被$n$除的余数是1。即$ab = 1 (mod \space n)$, 称$b$为a的模反元素。

## RSA 加密
###  (1) 生成公钥和私钥
- 随机选择两个不相等的质数$p$和$q$
$p=61, q=53$
- 计算$p$和$q$的乘积
$n=61\times 53 = 3233$
`3233`用二进制表示为`110010100001`,共12位，表示12位加密。
- 计算欧拉函数
$\varphi(3233) = \varphi(61) \times \varphi(53) = 60 * 52 = 3120$
- 随机选择整数$e$, 满足$1 < e < \varphi(n)$，并且$e$与$\varphi(n)$互质
$e=17$
- 计算$e$对于$\varphi(n)$的模反元素$d$
$ed - 1 = k\varphi(n)$, 选择一组解：$(d, k) = (2753,-15)$

在这里公钥为`(3233, 17)`，私钥为`(3233, 2753)`

### （2）加密和解密
- 使用公钥(n, e)加密
对于待加密的数值$m$, $m^e = c(mod \space n)$, 若`m=65`, 则`c=2790`
- 使用私钥(n, d)解密
$c^d ≡ m (mod \space n)$, 如果`c=2790`, 则 $2790^{2753} = 65 (mod \space 3233)$

## 加密解密算法证明
需要正如如下的等式成立：
$$c^d = m (mod \space n) \tag{1}$$
有加密过程可知如下等式成立
$$m^e=c (mod \space n) \tag{2}$$
转换一下
$$c = m^e - k n \tag{3}$$
将$c$带入$(1)$，我们要证明如下等式
$$(m^e - kn)^d = m (mod \space n) \tag{4}$$
因为等式左边包含了$n$项，也就是只需要证明 $m^{ed} = m (mod \space n)$成立，又因为$d$为$(e, \varphi (n))$的模反元素，因此$ed=h\varphi(n) + 1$， 也就是我们要证明如下等式成立
$$m^{h\varphi(n)+1} = m (mod \space n) \tag{5}$$
### 如果`m`和`n`互质
根据欧拉定理: $m^{\varphi(n)} = 1 (mod \space n)$, 带入
$$m^{h\varphi(n)+1}=(m^{\varphi(n)})^h \times m = (k'n+1)^h \times m $$
该式与$n$取模等于$m$成立。
### 如果`m`和`n`不是互质关系
因为 $n=p\times q$ 为两质数乘积，所以$m=kq$或者$m=kp$,以$m=kp$为例，其中$kp$和$q$必然互质，根据欧拉定理 $(kp)^{q-1} =1 (mod \space q)$ 进一步得到$((kp)^{q-1})^{h(p-1)} \times kp = kp (mod \space q) \rightarrow (kp)^{h(p-1)(q-1)+1} = kp (mod \space q)$
而又因为: $ed -1 = h\varphi(n)=h(p-1)(q-1)$，所以推导出
$$(kp)^{ed}=kp(mod \space q) \tag {6}$$
将其改写成
$$(kp)^{ed}=kp + tq \tag{7}$$
左边是$p$倍数，因此$t$必定为$p$被数，即$t=t'p$，所以上述式转换为
$$(kp)^{ed} = t'pq + kp \tag{8}$$
而又因为$m=kp, n=pq$，所以：
$$m^{ed}=m (mod \space n)$$


# 2 Review
[What every programmer should know about memory, Part I and II](https://lwn.net/Articles/250967/)
这一些列文章介绍了关于内存的基本知识，尤其是对程序员而言需要了解内容，整个系列文章主要分为以下几点：
- 内存物理结构
- CPU缓存
- NUMA系统
- 程序员需要关注-缓存优化
- 程序员需要关注-多线程优化
- 内存性能工具
- 将来的技术
## 2.1 硬件结构
![](./_image/2018-10-21-12-17-29.jpg)
上图为常见的个人电脑硬件架构，CPU通过前端总线(Front Side Bus, FSB)与北桥相连(Northbridge), 北桥通过内存控制器与内存相连；北桥通过南桥(SouthBridge)与其余系统通信。
## 2.2 RAM硬件结构
RAM(Random Access Memory)主要分为SRAM(static RAM 或者 synchronous RAM)和DRAM(dynamic RAM)
### 2.2.1 SRAM
![](./_image/2018-10-21-12-32-47.jpg)
上图为`SRAM`的电路结构，包含了6个晶体管，包含了两个稳定状态，不需要刷新电路。特点如下：
- 电路复杂，成本高;
- 访问速度快;
- 状态持续，不需要刷新电路。
### 2.2.2 DRAM

![](./_image/2018-10-21-12-36-48.jpg)
上图为`DRAM`的电路结构，只包含1个晶体管和1个电容。由于电容的存在，需要定时刷新电路。特点如下：
- 电路简单，成本低;
- 访问速度慢;
- 状态不持续，需要刷新电路。

# 2.3 高速缓存(CPU cache)
由于CPU和内存之间存在速度不匹配，需要使用高速缓存作为中间件。由于高速缓存容量需求小，要求速度快。一般而言高速缓存使用`SRAM`，一般的架构如下：

![](./_image/2018-10-21-12-46-42.jpg)
`L1`,`L2`和`L3`缓存层级容量逐渐增大，并且速度也减低。现代程序使用分段技术，数据部分和代码分别存在不同的段中，因此对`L1`缓存拆分成`L1d Cache`和`L1i Cache`分别缓存数据(data)和指令(instruction)。
**时间局部性和空间局部性**
高速缓存有两个是非常重要的概念：时间局部性(`temporal locality`)和空间局部性(`spatial locality`)
- *时间局部性*:  一个内存被使用过后，有很大的可能在不久的将来会被再次使用。
- *空间局部性*:  一个数据块包含很多数据对象，在访问其中一个数据对象后，很大可能访问数据块中的其他对象。

![](./_image/2018-10-21-13-12-23.jpg)
高速缓存整个结构如下，总共有个$2^S$组缓存，每一组共有$E$行缓存，每一行有$2^b$个缓存字节。
那么对于每一个内存地址都可以映射到该缓存中：
![](./_image/2018-10-21-13-19-35.jpg)
对于每一个内存地址`m`位，划分为三个部分，分别对应上述高速缓存的各个部分。这样每一个内存地址都可以缓存，而且满足了时间局部性和空间局部性要求。
# 3 Tip
在工作中经常需要登录跳板机进入到集群中容器中查看日志等相关操作，但是每次都需要进行用户名或者密码的验证，而且每次输入密码都非常繁琐，那可以编写脚本自动登录跳板机以及输入密码，所以考虑`bash`命令中的`expect`，该命令可以让脚本与交互式程序进行对话。具体可以查看[man expect](https://linux.die.net/man/1/expect)
```shell:n
#!/usr/bin/expect -f 
spawn ssh -p 2222 feng.gao@daoker.dui.ai
expect "feng.gao@daoker.dui.ai's password:"
send "my-daoker-password\r"
interact
```
从脚本代码中，首先使用`spwan`调用`ssh`登录跳板机，使用`expcect`期待给出的请求，然后使用`send`命令输入密码，注意最后加上`\r`表示回车，最后使用`interact`表示进入交互环境。
# 4 Share
在工作中，充分利用笔记等相关应用工作，将可能涉及到问题和看法记录下来，并考虑清楚，在做工作之前充分考虑到所有问题。