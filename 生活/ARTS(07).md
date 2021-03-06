---
date: 2018-11-26
status: public
tags: ARTS
title: ARTS(07)
---

# 1 Algorithm
[八皇后问题](https://en.wikipedia.org/wiki/Eight_queens_puzzle)是一道经典的算法题目，有多种解决问题的思路，甚至知乎上出现出现了[如何使用10行代码完成八皇后](https://www.zhihu.com/question/28543312)问题。
基本的解决问题的思路就是回溯法，不断的尝试每一行的皇后的位置，如果改行无可行的位置的，则回溯上一行皇后的位置。如果上一行皇后的位置也无法放置，则继续回溯，直至第一行。如有甚者，则算法结束。如果到达最后一行，如果位置合法，则表示一个完整的解决方案。
```go
package main

import "fmt"

var solutionCount = 0

// Queue for queue's position
type Queue struct {
	X int
	Y int
}

func abs(x int) int {
	if x < 0 {
		return -x
	}
	return x
}

func (q Queue) string() string {
	return fmt.Sprintf("{%d, %d}", q.X, q.Y)
}

func (q Queue) conflict(other Queue) bool {
	if q.X == other.X || q.Y == other.Y || abs(q.X-other.X) == abs(q.Y-other.Y) {
		return true
	} else {
		return false
	}
}

// QueueProblem solving
func QueueProblem(n int) {
	queues := make([]Queue, n, n)
	for idx := range queues {
		queues[idx] = Queue{idx, -1}
	}
	i := 0
	for i >= 0 {
		if i == n {
			printQueues(queues, n)
			i--
		}
		queues[i].Y++
		if queues[i].Y >= n {
			queues[i] = Queue{i, -1}
			i--
		} else {
			if satisfied(queues, i) {
				i++
			}
		}
	}
}

func satisfied(queues []Queue, n int) bool {
	for i := 0; i < n; i++ {
		if queues[i].conflict(queues[n]) {
			return false
		}
	}
	return true
}

func printQueues(queues []Queue, n int) {
	solutionCount++
	fmt.Println("position")
	for i := 0; i < n; i++ {
		fmt.Println(queues[i].string())
	}
}

func main() {
	QueueProblem(8)
	fmt.Printf("solution Count: %d\n", solutionCount)
}
```
# 2 Review
[Return in Go and C#](https://roberto.selbach.ca/returns-in-go-and-c/)
在`Go`语言用户中，大家经常抱怨`Go`的异常处理机制，但是我却认为它是`Go`语言中最有趣的地方。我在这不着重讨论错误处理机制，而是`Go`语言中的多返回值的特性。
举例来讲，`Go`中常见的代码模式如下：
```go
func Divide(a, b float64)(float64, error){
    if b==0{
        return 0.0, errors.New("divide by zero")
    }
    return a / b, nil
}
```
所以调用者会这样做
```go
result, err := Divide(x, y)
if err != nil {
    // do err handling
}
```
一些人不喜欢这样，但是我非常喜欢这种方式，因为它能够表达出更清晰的意图。在`C#`中没有多个返回值，那么我们是怎么做的呢？
- ** 抛出异常**
```csharp
public SomeObject GetObjectById(int id){
    if (!SomeObjectRepo.Has(id))
        throw new ArgumentOutOfRangeException(nameof(id));
    //...
}
//...
try{
    var obj = GetObjectById(1);
    // do tome thing obj
}
catch(ArgumentOutOfRangeException ex ){
    // error handling
}
```
但是控制流被`try-catch`打断了，有些变量被`try-catch`的作用域切割开来，需要在进入`try`之前定义变量。
- **返回null**
```csharp
public SomeObject GetObjectById(int id){
    if (!SomeObjectRepo.Has(id))
        return null
    // go get the object
}
//...
var obj = GetObjectById(1);
if (obj == null){
    // do error handling
}
```
这个方法虽然看上去好多了，但是还是有严重的问题，你没有得到任何错误的信息。而且对于返回是值类型的，`null`是无法返回的。当然也可以返回诸如`int?`, `bool?`等等。
-  **泛型返回值**
```csharp
public class Result<T>
{
    public T Value {get;protected set;}
    public Exception Exception {get; protected set;}

    public bool IsError => Exception != null;

    public Result() : this(default(T)) {}
    public Result(T value)
    {
        Value = value;
    }

    public static Result<T> MakeError(Exception exception)
    {
        return new Result<T>
        {
            Value = default(T),
            Exception = exception
        };
    }
}
```
有了通用的结果，就可以这样使用
```csharp
public Result<int> Divide(int a, int b)
{
    if (b == 0)
    {
        return Result<int>.MakeError(new DivideByZeroException());
    }
    return new Result<int>(a / b);
}
//...
var res = Divide(8, 4);
if (res.IsError)
{
    // do error handling, e.g.
    throw res.Exception;
}
// do something with res.Value (2)
```
代码看上去非常丑陋，接下来就能看到在`C# 7.0`中如何做到。
- **语法糖**
```csharp
public (int, Exception) Divide(int a, int b)
{
    if (b == 0)
        return (0, new DivideByZeroException());

    return (a / b, null);
}
// ...
var (res, err) = Divide(1, 2);
if (err != null) 
{
    // do error handling
}
```
现在代码和`Go`非常相似了，实际上就是利用了返回值为`tuple`的特性，但是`Go`语言的方式却更加优雅
# 3 Tips
使用`golang`中的`pprof`工具进行性能分析，首先对于`web`应用程序，添加
```go
import _"net/http/pprof"
```
可以在`localhost:port/debug/pprof`中查看性能分析， 选择`pprof`选线，可以对CPU进行采样分析，经过一段时间，生成`pprof`文件，使用如下命令可以生成`svg`文件
```
go tool pprof ./proff > pprof.svg
```
使用浏览器打开该文件，可以查看每个函数消耗时间。

![](./_image/2018-11-27-22-48-20.jpg)
颜色越深，表明该函数消耗时间越长。
# 4 Share
同样一件事情，不同的角度会有不同的看法和观点，如果我们觉得不同的说法都有道理，那么就需要在平时从不同的角度认真思考，形成自己的价值判断标准，才能不会产生价值观波动。