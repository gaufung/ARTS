---
date: 2018-11-05
status: public
tags: ARTS
title: ARTS(04)
---
# 1 Algorithm
「编程之美」中`中国象棋将帅问题`，在中国象棋中将帅不能出现在同一个纵轴序号上，要求输出所有非法的将帅位置，但是要求使用一个字节空间。
## 1.1 分析
根据中国象棋规则，将帅所有可能的位置总共有9种，简单粗暴的算法非常简单，只需要依次遍历将帅的所有组合的位置，如果不属于同一纵轴，则属于合法位置。
但是限制了只使用一个字节，那么就需要考虑使用字节则比特位`bit`来存储位置。一个字节有8个bit位，可以表示$2^8=256$种不同位置。但是在这里有将帅两个位置，所以考虑各使用4个bit表示，共可以表示$2^4=16$也是可以表示9个位置。

## 1.2 比特位操作
`AND(&)`, `OR(|)` 和 `XOR(^)` 是比特位操作常用的运算符，针对特殊的字节有一下功能
- `0xf0`与其他字节`AND`操作，可以将高四位的比特取出，低四位的比特位为`0`；同理`0x0f`取出低四位比特，高四位设置为`0`；
- `0x00`与其他字节`OR`操作，可以其他字节的比特位赋给`0x00`; 同理`0xff`与其他字节`OR`操作，保持不变；
    - `0xf0`与其他字节`OR`操作，高四位比特不变，低四位比特为其他字节低四位比特；
    - `0x0f`与其他字节`OR`操作，低四位比特不变，高四位比特为其他字节的高四位比特；
- `0xff`与其他字节`XOR`操作，将其他字节的所有比特位取反。

## 1.3 移位操作
移位操作是字节操作中常见的操作，分为`左移`和`右移`两种，但是程序中，针对无符号和有符号有两种不同的方式形式：
**无符号**
对于无符号数值，左移的规则非常简单，溢出的比特位丢弃，低位使用零补齐；右移规则同样，低位比特位丢弃，高位的比特位用零补齐；

**有符号**
对于有符号的数值，高位为负权重，为: $-2^{n-1}$，其余各位的权重为: $2^{m}, (0\le m < n)$。对于左移规则和无符号左移规则一样，溢出的高位丢弃；右移规则中溢出比特丢弃，高位用原先高位比特补齐。

## 1.3 实现
有了上面的基础，可以完成一些基础性工作，比如如何获取和设置不同位的值。
```go
const halfBitLength uint8 = 4 // bit legnth
const fullmask uint8 = 0xff   // 11111111
const lMask uint8 = 0xf0      // 11110000
const rMask uint8 = 0x0f      // 00001111
const gridWidth = 3           //grid width

func rightSet(b, n uint8) uint8 {
	return (lMask & b) | n
}
func leftSet(b, n uint8) uint8 {
	return (rMask & b) | (n << halfBitLength)
}

func rightGet(b uint8) uint8 {
	return rMask & b
}

func leftGet(b uint8) uint8 {
	return (lMask & b) >> halfBitLength
}
```
有了基础性工作，输出合法的将帅位置就非常简单了：
```go
func main() {
	var b uint8 = 0x00
	for b = leftSet(b, 1); leftGet(b) <= gridWidth*gridWidth; b = leftSet(b, leftGet(b)+1) {
		for b = rightSet(b, 1); rightGet(b) <= gridWidth*gridWidth; b = rightSet(b, rightGet(b)+1) {
			if (leftGet(b) % gridWidth) != (rightGet(b) % gridWidth) {
				fmt.Printf("A = %d, B = %d\n", leftGet(b), rightGet(b))
			}
		}
	}
}
```

# 2 Review 
[The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
这是一篇关于`ASCII`, `Unicode`, `UTF-8`等相关的博文，作者是大牛 Joel Spolsky， 他创办了`StackOverflow`等网站，也出版过`软件随想录`等书籍。
## 2.1 ASCII
在`Unix`创建那段时间，计算机还是在美国人广泛使用，英文共大小写52个字符，再加上一些符号和控制符，总的数量还是非常少的。因为使用一个字节(8bit)足以保存所有使用的字母和符号。事实上只有`32`到`127`表示为相关字符，小于`32`的是不可打印的字符，大于`127`还被保留着。
后来计算机被传递到欧洲，欧洲很多国家的语言包含了更多字符，没有办法在原先的字符集上使用，所以每个国家或者地区的OEM制造商充分使用了剩下的`128-255`的字符，用来表示该语言的特殊字符。这样导致了一个问题，如果我发送一个单词: `résumés`，那么到另外一个国家就会变成 `rגsumגs`, 这是应为`é`和`ג`共用了同一个数值表示。每个语言都有自己的表示方式，后来将每一个语言的表示方式汇总起来，也叫做`code pages`，在表示处理不同的语言的时候，取出该语言对应的`code page`，然后进行对应解码。
## 2.2 Unicode
对于拉丁语言为主的语言，高位的数值数量足以用来表示，但是对于东亚国家来说，有成千上万的字符，`ASCII`表示剩下的高位明显不够了，当时采用叫`DBCS` (double byte character set)的方式来处理，有些字符用一个字节保存，对于东亚的语言字符使用两个字符表示。但是随着互联网的发展，不过国家地区的交流越来多，这个字符表示法的弊端越发明显，因此`Unicode`被提出来了。
在`Unicode`中，每一个字符使用`16bit`，也就是2个字节表示。表示的值叫作`code point`，不过要记住，这个只是概念上的形式，至于如何在内存或者磁盘中表示又是另外一回事。`Unicode`采用形如`U+0639`形式表示，`U+`表示为`Unicode`，而`0639`是十六进制的形式，转换成二进制就是`0000 0110 0011 10001`。
## 2.3 Encoding
比如字符串: `**Hello**`, 在 `Unicode`中表示为
[`U+0048 U+0065 U+006C U+006C U+006F`]
在早期的编码方案中，使用使用`Unicode`的`code point`保存在内存或者磁盘中，所以保存方式如下：
[`00 48 00 65 00 6C 00 6C 00 6F`]
但是也可以这样表示：
[`48 00 65 00 6C 00 6C 00 6F 00`]
这就涉及`大端`和`小端`的问题，是高位在前呢，还会低位在前？这两种方式都是可以的，这些取决于CPU或者计算机架构。
当然这个方案完美的解决了上述的问题，但是带来了原先`ASCII`编码的字符串大量的空间浪费，因为他们所有的第高位字节都是零，也就是说有一半的空间是浪费的。而且之前的保存的字符串也作废了。所以`UTF-8`编码方案被提出来了，原先的`ASCII`字符仍然是一个字节表示，而其余字符由两个、三个字符组成，判断规则根据比特位值得范围。
- 0000~007F：一个字节；
- 0080~07FF：两个字节；
- 0800~FFFF：三个字节。
汉字在UTF-8中表示为三个字节。

## 2.4 Others
穿传统使用2个字节表示`Unicode`叫做`UCS-2`，也有使用4个字节表示`Unicode`的方案，叫做`UCS-4`。
如果使用某种编码方案，计算得到的`code point`不存在，就会得到一个问号：`?`，有时候也会得到:`�`。
在网络服务器中，在`Response`发送之前，会在HTTP请求头中添加
[`Content-Type: text/plain; charset="UTF-8"`]

告诉浏览器如何进行解码。如果在请求头中不包含这个信息，那么会在网页元数据中添加
[`<meta http-equiv="Content-Type" content="text/html; charset=utf-8">`]
如果这个信息也没有，浏览器会猜测使用的编码方式，根据每个语言中字符出现的频率，根据特定的编码出现的频数直方统计图，获得相应的编码方案。
# 3 Tips
使用`ls`命令，使用`-t`参数，可以按照时间排序。
# 4 Share
在程序设计中，接口设计是一个非常重要的工作，对于接口设计有一个原则: ` Postel’s Law`
> conservative in what you emit and liberal in what you accept

翻译过来就是：
> 对你输出的东西要按照原则，对于你接受的东西要包容

这个原则怎么理解呢？以`go`语言为例，查看下面的代码实现方式
```go
func New1() *io.writer {
    //...
}
func New2() *os.File {
    //...
}
```
`New1`和`New2`两个函数哪一个比较好，它们不同点在于返回值。按照`Postel's Law`原则，我们只需要返回实现`Write`方法的接口即可，所以`New1`方法更好。
那么对于下面两个方法
```go
func Func1(*io.Writer){
    //...
}
func Func2( *os.File){
    //...
}
```
`Func1`和`Func2`函数不同在于接受的参数不一样，同样按照`Poststel's Law`原则，`Func1`函数更能接受更多的参数，包容性更好。