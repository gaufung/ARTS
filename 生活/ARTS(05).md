---
date: 2018-11-11
status: public
tags: ARTS
title: ARTS(05)
---
# 1 Algorithm
**长胜将军问题**：假设有`n`个棋子，`A`和`B`两个棋手分别依次从中取出`1-m`个棋子，取到最后一个棋子的棋手胜出，那么给出一个策略，能够保证`A`或者`B`永远胜出。
为了问题形象化，我们假设`n=22`, `m=4`，`A`首先取棋子，提供的策略保证`A`始终胜利，同时也要假设`B`棋手是个“大傻子”，选棋子数量都是随机的。
## 1.1 策略分析
当`A`取完全棋子后，如果还剩下`5`个棋子，那么不管`B`取的方式，`A`总能获胜。那么问题转换为共`22-5=17`个棋子，在`A`能够取完全部棋子。那么问题就变成递归的执行，最后变成最后`17-5-5-5=2`棋子，`A`第`2`个取棋子，在后边每一轮中，保证和`B`取的棋子数量之和为`5`即可。
如果最后棋子是一颗炸弹，谁最后一个抽中就失败。策略就要保证当`A`取完后，只剩下`1`棋子，那么问题就转换为有`22-1=21`个棋子，保证`A`能够取到最后一个棋子，问题就变成上述的情况，`21-5-5-5-5-5=2`，那么`A`首先取出`2`个棋子，那么接下来每次和`B`取出的棋子数量和为`5`即可。
### 1.2 分析过程
如果`A`是首先取棋子，那么要保证在最后一轮之前，还剩下`1+m`个棋子，如果最后一个取棋子失败，那么保证最后剩下`1`棋子，那么剩下的棋子`n-(1+m)`或者`n-1`与`m+1`的模则是`A`第一轮取棋子的数量，模必定在`0-m`之间，如果为`0`，那么不能保证能够取胜。
### 1.3 实现
使用`go`语言中的通道`channel`可以很形象的模拟取棋子的过程，`A`和`B`两个选择的取棋子的过程用两个通道模拟，`A`棋手通过通过通道得到剩余的棋子数量，然后计算得到自己的应该取的棋子；然后将剩余的棋子数量放回另外一个通道中；而`B`棋手从该通道获取剩余的棋子，`随机`取棋子，然后写入到第一个通道。一旦获取的棋子的数量为零，则表明失败。
```go
func playerA(n, m int, in <-chan int, out chan<- int) {
	left := (n - (m + 1))
	firstPick := left % (m + 1)
	if firstPick == 0 {
		fmt.Print("player A losses the game\n")
		close(out)
		return
	}
	left = n - firstPick
	out <- left
	for nowLeft := range in {
		if nowLeft == 0 {
			fmt.Print("player A losses the game\n")
			close(out)
			break
		}
		oppoentPick := left - nowLeft
		pick := (m + 1) - oppoentPick
		left = nowLeft - pick
		out <- left
	}
}
func playerB(m int, in <-chan int, out chan<- int) {
	rand.Seed(time.Now().UnixNano())
	for nowLeft := range in {
		if nowLeft == 0 {
			fmt.Print("player B losses the game\n")
			close(out)
			break
		}
		if nowLeft < m {
			out <- 0
		} else {
			pick := 1 + rand.Intn(m)
			left := nowLeft - pick
			out <- left
		}
	}
}
func main() {
	in, out := make(chan int), make(chan int)
	n, m := 21, 4
	go playerA(n, m, in, out)
	playerB(m, out, in)
}
```

# 2 Review
[Practical Go: Real world advice for writing maintainable Go programs](https://dave.cheney.net/practical-go/presentations/qcon-china.html)
![](./_image/Practical Go.jpg)
这篇文章对于如何构建健壮`Go`项目提出很好的建议。
## 2.1 Guiding Principles
对于程序来讲，最重要的三个原则就是`Simplicity`，`Readability`和`Productivity`，而对于Go语言中`性能`或者`并发`并不是最重要的。
## 2.2 标识符
首先标识符命名要`清晰`而不是`短小`，有一下三点要求：
- 简洁，同时也要有最小的噪声；
- 可表述性，可以表达出意图；
- 可预测性，循序一般的程序惯例。
### 2.2.1 标识符长度
标识符长度越小，其影响的范围越小；反之亦然。
- 命名不要包含类型；
- 常量命令应该描述其值，而不是怎么用；
- 单个字母`i,j,k`在循环中；单个词用在参数和返回值；多个字用在函数和包级别声明中
- 单一字用在方法，接口和包命名。
- 对于调用者，包名字也是命名的一部分。

## 2.2.2 上下文也是关键
对于下面两种方式
```go
for index:=0; index < len(s); index++ {
    //..
}
for i:=0; i < len(s); i++ {
    //...
}
```
两者都是循环遍历的环境中，两种方式都是可以的。但是对于下面两种描述
```go
func (s *SNMP) Fetch(oid []int, index int) (int, error)
func (s *SNMP) Fetch(o []int, i int) (int, error)
```
第一种显然是更好的命名方式，因为调用者并不知道序列的索引的上下文环境。

### 2.2.3 统一的命名风格
统一的命令风格可以减少读者的认知难度。
- `i,j,k`用在循环中；
- `n`通常表示数量或者累加器；
- `v`通常用在泛型的函数；
- `k`表示键；
- `s`用来表示字符串。

### 2.2.4 统一的声明方式
- 如果声明，但没有初始化，使用`var`形式
- 如果声明，并且进行初始化，使用`:=`形式

### 2.2.5 团队成员
如果团队有约束规范，按照团队规范。

## 2.3 Comment
> 好的代码有很多注释，而差的代码需要很多注释

- 注释用来解释做什么，一般为公共的接口；
- 注释用来怎么做，一般在方法内部；
- 注释用来解释为什么这么做，特殊情况；
- 对于变量和常量，注释应该用来描述它内容，而不是目的；
- 对于公共接口，一定要编写注释；
- 对于差的代码，不要写注释，重写它，如果目前条件不允许，保留为技术债。
- 对于代码块，不要写注释，重构它。

## 2.4 Package Desigin
### 2.4.1 包命名
包的命名也是非常重要的，而且最好是单个词。在选择名称的时候，思考这样的问题：**这个包会提供什么服务？**，而不是 *哪些类型应该放在这个包中？* 
- 包的名称应该独一无二；
- 避免`base`，`common`或者`util`等名称；
- 包的名称也是标识符的一部分；
如果`Get`函数在包`net/http`中，那么在使用的时候就是`http.Get`.
### 2.4.2 提前退出
避免箭头型代码，对于不满足条件的情况，提前`return`返回。
```go
// early return
func (b *Buffer) UnreadRune() error {
	if b.lastRead <= opInvalid {
		return errors.New("bytes.Buffer: UnreadRune: previous operation was not a successful ReadRune")
	}
	if b.off >= int(b.lastRead) {
		b.off -= int(b.lastRead)
	}
	b.lastRead = opInvalid
	return nil
}
// without guard cluase
func (b *Buffer) UnreadRune() error {
	if b.lastRead > opInvalid {
		if b.off >= int(b.lastRead) {
			b.off -= int(b.lastRead)
		}
		b.lastRead = opInvalid
		return nil
	}
	return errors.New("bytes.Buffer: UnreadRune: previous operation was not a successful ReadRune")
}
```
###  2.4.2 让零值有意义
在`go`中如果只做了声明，没有初始化，对于数值类型标记为`0`，对于应用类型为`nil`，所以对于`零值`也要考虑其作用，如果是类型的，在`recevier`中，可以接受对象为空值判断。
```go
type Config struct {
	path string
}

func (c *Config) Path() string {
	if c == nil {
		return "/usr/home"
	}
	return c.path
}
type Config struct {
	path string
}

func (c *Config) Path() string {
	if c == nil {
		return "/usr/home"
	}
	return c.path
}

func main() {
	var c1 *Config
	var c2 = &Config{
		path: "/export",
	}
	fmt.Println(c1.Path(), c2.Path())
}
```
### 2.4.3 避免包级别状态
对于编写可维护的程序，应该各个包之间是低耦合的。在Go降低耦合程序有两种方法：
- 使用接口来描述行为，函数和方法来实现；
- 避免使用全局状态

## 2.5 Project Structure
如果项目是一个库`library`，那么它应该只做一件事；如果是一个应用程序，那么应该有一个`cmd/contour`包，用来不同的目的。
## 2.5.1 考虑更少、更大的包
从其他语言中转过来的开发人员，常常过度使用包，`go`只提供了`public`和`private`两种访问级别，所以在组织的过程中需要考虑使用更少但是更大的包。
> java中的包(package)相当于go中的单个`.go`源文件。 而Go中的包则相当于整个`maven`的模块或者`.Net`中的`assembly`。

### 2.5.2 组织代码至文件中
- 开始每一个包中只有一个`.go`文件，并且与文件夹同名；
- 随着项目增加，将不同责任的代码划分到不痛的文件中；
- 如果多个文件包含相同的`import`声明，则考虑将他们合并起来。
- 不同的文件应该为包中不同的领域负责。

### 2.5.3包中的测试
对于每一个`x.go`文件，那么`x_test.go`就是该文件的单元测试，在编译的时候，这个是不包含在其中的，这就是内部`internal`测试；如果包的命名为`x_test`那么，那么别的包在使用的时候就是外部`external`测试，`Exmaple`测试函数就应该在外部测试。

### 2.5.4 使用`internal`包来减少公共API
如果有`../a/b/c/inernal/d/e/f`组织结构，那么`../a/b/c`则可以被导出，而`d/e/f`则不能被导出。

### 2.5.5 使`main.go`尽可能小
`main`函数应该尽可能小，只做一些初始化的工作，如连接数据库，解析命令行参数等等。

## 2.6 API Design
###2.6.1 设计API应该保证很难用错
如果函数有多个相同类型的参数，那么参数的顺序是否有影响
```go
func CopyFile(to, from string) error
```
在`CopyFile`函数中，如果不查看手册，很难知道那个参数是拷贝源，另一个是拷贝目的。如果API是这么设计
```go
type Source string
func (src Source) CopyTo(dest string) error {
	return CopyFile(dest, string(src))
}
```
这种方式大大较少API被误用的可能性。
### 2.6.2 让函数定义为行为所需要的
如果函数
```go
func Save(f *os.File, doc *Document) error
```
在这个函数API设计中`*os.File`参数包含了`Save`方法行为需要的太多的功能。因此只需要`io.Writer`即可，所以API设计
```go
func Save(w io.Writer, doc *Document) error
```
## 2.7 Error Handling
### 2.7.1 通过消除错误来消除错误处理
比如下面的例子用来举例，如果使用常规的处理方法
```go
func CountLines(r io.Reader) (int, error) {
	var (
		br    = bufio.NewReader(r)
		lines int
		err   error
	)

	for {
		_, err = br.ReadString('\n')
		lines++
		if err != nil {
			break
		}
	}
	if err != io.EOF {
		return 0, err
	}
	return lines, nil
}
```
在整个代码中，包含了大量的处理过程，`ReadString`错误检查，`err!=io.EOF`文件末尾检查。通过换一种方式来进行处理：
```go
func CountLines(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	lines := 0
	for sc.Scan() {
		lines++
	}
	return lines, sc.Err()
}
```
在这里使用了`Scanner`结构来处理，`Scan()`能够匹配默认匹配`\n`模式，如果匹配成功，返回`true`，否则返回`false`。而`Err()`返回错误原因，如果没有错误，则为`nil`。通过修改返回值为布尔型，消除了错误。
### 2.7.2 使用`github.com/pkg/errors`包装错误
使用`github.com/pkg/errors`来包装错误，获取完整的信息
```go
func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, errors.Wrap(err, "open failed")
	}
	defer f.Close()

	buf, err := ioutil.ReadAll(f)
	if err != nil {
		return nil, errors.Wrap(err, "read failed")
	}
	return buf, nil
}

func ReadConfig() ([]byte, error) {
	home := os.Getenv("HOME")
	config, err := ReadFile(filepath.Join(home, ".settings.xml"))
	return config, errors.WithMessage(err, "could not read config")
}

func main() {
	_, err := ReadConfig()
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```
将会输出
```
could not read config: open failed: open /Users/dfc/.settings.xml: no such file or directory
```
# 3 Tips

# 4 Share
在程序设计模块中，需要考虑的是这个模块的功能，也就是它需要做什么；而不是怎么实现，实现乃是技术上的细节，而不是最重要的，这样做才能做到最大程度的解耦。