---
title: golang中big包源码阅读——从RSA算法说起
status: public
date: 2018-10-28
---
# 1 Golang中RSA加密算法实现
## 1.1 RSA加密算法基础
RSA加密算法属于非对称加密算法，属于网络的基础安全算法。阮一峰的博文：[RSA算法原理(一)](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)和[RSA算法原理(二)](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)，非常通俗易懂。在这里简单的归纳总结一下，整个算法分为三个步骤，分别为：生成公钥和密钥；发送方使用公钥生成密文；接收方使用密钥解密。
**生成公钥和私钥**
- 选择两个较大的质数 $p$ 和 $q$ ;
- 计算 $p$ 和 $q$ 的乘积 $n = p \times q$ ；
- 随机选择整数 $e$, 保证 $1 < e < \varphi(n)$ 并且 $ e, \varphi(n)$ 互质，其中 $\varphi(n)$ 为 $n$ 的欧拉函数值；
- 方程 $e\times d - 1 = k \times \varphi(n)$的一组解：$(d, k)$；
- 公钥：$(n, e)$；私钥： $(n, d)$
**公钥加密**
对于待加密的数值：$m$, 那么密文: $c = m^{e} \space mod \space n$。

**私钥解密**
通过$(n, d)$和密文$c$，计算得到密文: $m = c^d \space mod \space n$。
## 1.2 算法优化
在解密的算法中，关键点在于计算$c^d$和对于$n$取模，但是通常情况下，该数是非常大的，因此计算是非常耗时操作。所以对于RSA算法解密的过程有简化的方法。
**中国剩余定理**
在*孙子算经*中有下面这么一段话
> 有物不知其数，三三数之剩二，五五数之剩三，七七数之剩二。问物几何？ 

换成RSA中就是这样描述：$p$和$q$是两个素数，$n=p\times q$, 对于任意$(m_1, m_2), (0 \le m_21 <p, 0 \le m_2 < q)$, 必然存在唯一的整数$m, (0 \le m < n)$ 满足 $m_1 = m \space mod \space q, m_2 = m \space mod \space p$， 所以RSA解密算法中的$m = c^d \space mod \space n$, 可以分解为$m_1 = c^d \space mod \space p, m_2 = c^d \space mod \space q$, 然后再求得$m$。
对于$c^d \space mod p = \ldots = c^r \space mod \space p$, 其中$r$为$d$除$p-1$的余数， 即$r= d mod (p-1)$, 令$d_p =  d \space mod \space (p-1)$，同理$d_q = d \space mod (q-1)$。同时计算出$q_{inv} \times q= 1 \space mod \space p$。在预先计算出结果后，就可快速的解密
1. $m_1 = c ^{d_p} \space mod \space p$
2. $m_2 = c ^{d_q} \space mod \space q$
3. $h = (q_{inv} \times ((m_1-m_2) \space mod \space p)) \space mod \space p$
4. $m = m_2 + h * q$

## 1.3 多素数
之前讨论的都是两个素数生成加密算法，为了保证$n$的位数，可以选择超过两个的素数，$p, q, r_1, r_2 \ldots, r_n$，生成公钥和私钥的过程和之前一样，加密和解密的直接算法也是同样的。同样可以使用算法的优化算法。
## 1.2 Golang中实现方式
在`Golang`中实现了RSA加密算法：`src/crypto/rsa/rsa.go`文件中实现了RSA算法。该算法实现上述讨论的内容，但是除此之外，还处理可能出来的问题。如果$m^e$的值比$n$还小，那么$c = m^e$，所以根据$c$很容易的计算出$m$，因此通常是增加$m$的值，使之与$n$接近，`PKCS1`和`OAEP`都是很好的方法，在这里不做重点讨论。
### 1.2.1 加密
公钥的数据结构：
```go
type PublicKey struct {
	N *big.Int // modulus
	E int      // public exponent
}
```
包含了公钥必须$n$和$e$，但是两个是不同的数据类型`big.Int`和`int`两种。加密过程也是非常简单
```go
func encrypt(c *big.Int, pub *PublicKey, m *big.Int) *big.Int {
	e := big.NewInt(int64(pub.E))
	c.Exp(m, e, pub.N)
	return c
}
```
其中`Exp`方法作用$c = m^e \space mod \space \text{pub.N} $
### 1.2.2 解密
私钥的数据结构
```go
type PrivateKey struct {
	PublicKey            // public part.
	D         *big.Int   // private exponent
	Primes    []*big.Int // prime factors of N, has >= 2 elements.
	// Precomputed contains precomputed values that speed up private
	// operations, if available.
	Precomputed PrecomputedValues
}
```
私钥结构包含(`embed`)了公钥的结构，也可以知道使用了多素数的计算的方式，并使用`PrecomputedValues`结构保存加速解密计算的值，具体信息如下：
```go
type PrecomputedValues struct {
	Dp, Dq *big.Int // D mod (P-1) (or mod Q-1)
	Qinv   *big.Int // Q^-1 mod P
	CRTValues []CRTValue
}
// 包含了中国余数定理的值
type CRTValue struct {
	Exp   *big.Int // D mod (prime-1).
	Coeff *big.Int // R·Coeff ≡ 1 mod Prime.
	R     *big.Int // product of primes prior to this (inc p and q).
}
```
其中`Dp`,`Dq`和`Qinv`是之前算法描述的预先计算的值，而`CRTValue`切片包含了使用中国余数定理所需要的值。
#### 1.2.2.1 生成私钥
```go
func GenerateKey(random io.Reader, bits int) (*PrivateKey, error) {
    // 生成只有两个2个素数的RSA
	return GenerateMultiPrimeKey(random, 2, bits)
}
func GenerateMultiPrimeKey(random io.Reader, nprimes int, bits int) (*PrivateKey, error){
    // 设置E的默认值为65537
    priv := new(PrivateKey)
    priv.E = 65537
NextSetOfPrimes:
    for {
        // 确定设置还需要的剩余的bit位
        todo := bits
        //生成需要需要的bit位的素数
        for i := 0; i < nprimes; i++ {
			var err error
			primes[i], err = rand.Prime(random, todo/(nprimes-i))
			if err != nil {
				return nil, err
			}
			todo -= primes[i].BitLen()
		}
		n := new(big.Int).Set(bigOne)
		// totient 保存 n 的欧拉函数值
		totient := new(big.Int).Set(bigOne)
		pminus1 := new(big.Int)
		for _, prime := range primes {
			n.Mul(n, prime)
			pminus1.Sub(prime, bigOne)
			totient.Mul(totient, pminus1)
		}
        	priv.D = new(big.Int)
		e := big.NewInt(int64(priv.E))
		// 根据E值计算出D值
		ok := priv.D.ModInverse(e, totient)
		//...
    }
    // 为解密过程中预先计算
    priv.Precompute()
	return priv, nil
}
```
在RSA中，公钥中默认为：$e=65537$，按照所需的素数的个数和生成$n$的位数生成素数和$d$，最后进行预先计算操作，以加快解密过程。
```go
func (priv *PrivateKey) Precompute() {
    //....
	priv.Precomputed.Dp = new(big.Int).Sub(priv.Primes[0], bigOne)
	priv.Precomputed.Dp.Mod(priv.D, priv.Precomputed.Dp)

	priv.Precomputed.Dq = new(big.Int).Sub(priv.Primes[1], bigOne)
	priv.Precomputed.Dq.Mod(priv.D, priv.Precomputed.Dq)

	priv.Precomputed.Qinv = new(big.Int).ModInverse(priv.Primes[1], priv.Primes[0])
	//...
}
```
对于两个素数的提前计算比较直观，对私钥中的`Precomputed`中的`Dp`,`Dq`和`Qinv`分别计算。
#### 1.2.2.2 解密
```go
func decrypt(random io.Reader, priv *PrivateKey, c *big.Int) (m *big.Int, err error) {
	//....
    if priv.Precomputed.Dp == nil {
		m = new(big.Int).Exp(c, priv.D, priv.N)
	} else {
		// We have the precalculated values needed for the CRT.
		m = new(big.Int).Exp(c, priv.Precomputed.Dp, priv.Primes[0])
		m2 := new(big.Int).Exp(c, priv.Precomputed.Dq, priv.Primes[1])
		m.Sub(m, m2)
		if m.Sign() < 0 {
			m.Add(m, priv.Primes[0])
		}
		m.Mul(m, priv.Precomputed.Qinv)
		m.Mod(m, priv.Primes[0])
		m.Mul(m, priv.Primes[1])
		m.Add(m, m2)
		//...
		}
	}
	//...
	return
}
```
如果没有提前计算，那么直接使用公式计算；如果进行已经提前计算值，则按照优化的算法依次计算。
# 2 Golang中Big包
由于`RSA`算法在实现过程中需要很大（位数很多）的数据，所以没有使用`int`、`int32`、`int64`等数据类型，而是使用`math.big`包中提供的`Int`类型。除了`Int`类型，还定义了`Rat`,`Float`等相关类型，由于`Go`不支持操作符重载，所以基本上运算使用`Add`, `Sub`等形式定义的，在类型方法中，返回值通常也是`receiver`，所以在使用过程中，不需要定义一些变量保存结果，直接使用链式调用即可。
## 2.1 类型
在`src/math/big`中，实现了整数`Int`，浮点数`Float`和有理数`Rat`三种使用到的数据类型。除此之外还有一些辅助类型和针对大数处理的函数。
### 2.1.1 Word (`src/math/big/arith.go`)
```go
type Word uint
```
`Word`类型是`uint`的别名，它代表了在`big`包中基本操作单元，其中包含了一些列基本的算术计算函数，除了`Word`之间的加减乘除计算；定义了`[]Word`和`Word`之间的加减乘除计算；定义了`[]Word`之间的加和减计算。
### 2.1.2 nat (`src/math/big/nat.go`)
```go
type nat []Word
```
`nat`是`[]Word`的别名，和整数表示形式一样，`nat`中每一个元素表示一位数字位，所以对于任意`nat`表示的任意数值`x`，都有：
$$x =x[n-1]\times B^{n-1} + x[n-2] \times B^{n-2} + \ldots + x[1]\times B + x[0]$$
其中`B`为`Word`表示值的基，通常为`1<<32`或者`1<<64`，取决于`uint`的类型是32位还是64位。除此之外，`nat`表示的值在最终的结果中，是不包含前面的零。
定义了`nat`之间的加、减、乘、除等操作，还定义了区间内的连乘、平方根、取模；也提供了`nat`池，达到重复使用的目的。
### 2.1.3 Int (`src/math/big/int.go`)
```go
type Int struct {
	neg bool // sign
	abs nat  // absolute value of the integer
}
```
`Int`类型定义包含了一个布尔型值`neg`，表示该值是正数还是负数；一个`nat`类型，表示该整数的绝对值。
除了定义常规的整数之间运算，还定义了诸如`int32`,`int64`等和`Int`之间互相转换；字符串和`Int`类型相互转换；`And`,`OR`,`NOT`等运算；最大公约数`GCD`,取模`MODE`和素数等相关的计算方法。
### 2.1.4 Rat(`src/math/big/rat.go`)
```go
type Rat struct {
	a, b Int
}
```
有理数$\frac{a}{b}$中的分子分母`a`和`b`为`Int`类型，提供了常规的算术运算；还有有`float32`, `float64`等相关转换操作。
### 2.1.4 Float(`src/math/big/rat.go`)
```go
type Float struct {
	prec uint32
	mode RoundingMode
	acc  Accuracy
	form form
	neg  bool
	mant nat
	exp  int32
}
```
浮点型数据表示方式：
$$sign \times mantissa \times 2 ^{exponent}$$
其中 $0.5 \le mantissa \le 1.0$, 而且$MinExp \le exponent \le MaxExp$。除此之外还包含以下三个变量：
- 精度(`precision`): 表示`mantissa`比特位表示值的最大值；
- 取值模式(`mode`): 表示将浮点值转换为`mantissa`表示时候取值模式，一般有`ToNearestEven`, `ToNearestAway`,`ToZero`等等；
- 准确度(`accuracy`):表示取舍值与真正值之间的差值，取值有三种：`Below`,`Exact`和`Above`。
在`Float`类型中的`form`内部使用，用来表示该浮点值是零值，无穷值还是有穷值。
`Float`定义的精度限制范围：
```go
const (
	MaxExp  = math.MaxInt32  // largest supported exponent
	MinExp  = math.MinInt32  // smallest supported exponent
	MaxPrec = math.MaxUint32 // largest (theoretically) supported precision; likely memory-limited
)
```
与 [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) 定义的浮点型方式稍微有点不同：`mant`是一个非零的有限值，`nat`切片通常保存`precision`要求的位数，但是如果后面都是`0`,那么`nat`舍弃这些零，如果`precision`不是`Word`长度的整数倍，那么就要在`mant[0]`后面补上`0`; 如果`x.mant=1`，也就是`mantissa=0.5`，将会做一些标准化，将`mantissa`进行左移操作，`exponent`部分会右移操作。统一的形式为
```
x                 form      neg      mant         exp
----------------------------------------------------------
±0                zero      sign     -            -
0 < |x| < +Inf    finite    sign     mantissa     exponent
±Inf              inf       sign     -            -
```
和其他类型一样，`Float`提供的大量计算的方法。

