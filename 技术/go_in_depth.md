---
date: 2019-03-19
status: public
tags: Knowledge
title: Go 语言学习笔记
---
# 1 类型
## 1.1 简短模式
- 定义变量，同时显示初始化
- 不能提供数据类型
- 只能用在函数内部

退化为赋值操作，当且仅当 `至少有一个新的变量被定义，切必须处于同一作用域`

## 1.2 常量
编译器能够计算出来的表达式，也有一作为常量
```go
const (
	ptrSize = unsafe.Sizeof(uintptr(0))
	strSize = len("hello, world!")
)
```
## 1.3 引用类型
预定义的引用类型 `slice, map, channel`，`new` 按照类型长度分配零值类型，返回指针，不关心内部构造和初始化方式；`make` 转换为目标类型专用的创建函数和指令，确保内存分配和相关属性初始化。

# 2 表达式
## 2.1 指针
- 并非所有的对象都可以取地址，但是变量总能正确返回
```go
m := map[string]int {"a": 1}
println(&m["a"]) //panic
```
- 零长度的（zero-size)对象地址与实现有关，不等于`nil`

## swith 语句
从上到下，从左到右匹配 `case` 语句，只有到最后才匹配 `default` 语句。

# 3 函数
## 3.1 函数定义
从函数返回局部变量的指针是安全的，编译器会使用逃逸分析(`escape analysis`) 来决定是否分配在堆上。
```go
func test() *int {
    a := 0x100
    return &a
}

func main(){
    var a *int = test()
    fmt.Println(a, *a)
}
```
## 3.1 参数
都是值拷贝传递（`pass-by-value`)，区别在于是拷贝目标对象，还是拷贝指针。
## 3.2 返回值命名
命令返回值让函数声明更加清晰，同事改善帮助文档。
## 3.2 闭包
闭包本质上是返回`funcval`对象，该对象包含了指向所引用的环境变量的指针。
## 3.3 延迟调用(`defer`)
执行步骤：
- 更新返回值
- 调用`defer`语句
- 执行`return`
`defer`包含了函数的注册、调用和额外的缓存开销。

# 4 数据
## 4.1 切片
`slice` 不是数组或者数组指针，它通过内部指针和相关属性应用数组片段
```c
struct Slice
{
  byte* array;
  uintgo len;
  uintgo cap;  
};
```
`reslice` 中 s[low:high:maxcap]

## 4.2 map 
map 因为扩张而重新哈希，因此 map 是 `not addressable`，通过类似 `m[1].name` 来修改是被禁止的
```go
type user struct { name string}
m := map[int]user {
    1: { "user1" },
}
m[1].name = "Tom" // error 
```
`map` 在迭代的过程中，也可以删除键值。

## 4.3 匿名字段
```go
type User struct {
    name string
}
type Manager struct {
    User
    title string
}
```
匿名字段是一种语法糖，创建了一个成员类型同名的字段。

# 5 方法
方法绑定对象实例，并且隐式将实例作为第一实参（`receiver`)，可以用实例 `value` 和 `pointer` 调用全部算法，编译器自动转换。
## 5.1 方法集
- 类型T方法集包含全部 receiver T 方法；
- 类型*T方法包含全部 receiver T + *T 方法；
- 类型S包含匿名字段 T， 则 S 方法包含包含 T 方法；
- 类型S 包含匿名字段 *T， 则 S方法集合包含 T + *T 方法；
- 不管嵌入T或者*T， *S 方法集合总包含 T + *T方法。
## 5.2 表达式
```
instance.method(args...)
<type>.func(instance, args ...)
```
# 6 接口
匿名接口可以作为变量类型或者结构成员
```go
type Tester struct {
    s interface {
        String() string
    }
}
type User struct {
    id int 
    name string
}
func (u *User) String() string {
    return fmt.Sprintf("user %d, %s", u.id, u.name)
}
func main(){
    t := Tester{&User{1, "Tom"}}
    fmt.Println(t.s.String())
}
```
## 6.1 执行机制
```c
struct Iface
{
    Itab* tab;
    void* data;
}
struct Itab
{
    InterfaceType* inter;
    Type* type;
    void (*fun[])(void);
}
```
接口表存储元数据的信息，包含接口类型，动态类型，已经实现接口的方法指针。
只有`tab` 和 `data` 都为 `nil` 的时候，接口才等于 `nil`。

# 7 并发
- 并发：逻辑上具备同时处理多个任务的能力
- 并行：物理上同一个时刻执行多个任务的能力

## 7.1 Goroutine
`runtime.GOMAXPROCS` 让调度器实现多线程多核并行，而不仅仅是并发；
`runtime.Goexit` 退出当前 `goroutine` 执行，并且执行已经注册的`defer` 函数调用；
`runtime.Gosched` 将当前 `goroutine` 暂停，放回队列等待下次执行。

## 7.2 资源泄漏
通道引发`goroutine leak`, `goroutine` 处于发送和接收阻塞状态，但是一直未被唤醒，垃圾回收并不回收此类资源。

## 7.3 同步
通道倾向于解决逻辑层次的并发处理
锁则为保护局部范围内的数据安全。
而且锁赋值将会导致机制失效，因此需要 `pointer-receiver`。 

# 8 包结构
`import ` 导入的参数是路径，而并非包名。

# 9 反射
Go 反射都是通过接口的信息获取
```go
func TypeOf(i interface{}) Type
func ValueOf(i interface{}) Value
```

## 9.1 类型
面对类型的时候，还需要区分 `Type` 和 `Kind`， 其中 `Type` 为静态类型，而 `Kind` 为基础类型
```go
type X int
func main(){
    var a X = 10
    t  := reflect.TypeOf(a)
    fmt.Println(t.Name(), t.Kind())
}
```
方法 `Elem` 返回指针，数组、数组、切片、字典(值)或者通道的基础类型，只有获得结构体指针的基础类型后，才能遍历它的字段。

## 9.2 值
想要修改对象的值， 必须使用指针。

