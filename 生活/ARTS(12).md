---
date: 2019-01-01
status: public
tags: ARTS
title: ARTS(12)
---
# 1 Algorithm
>给定 n 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。
> 

![](./_image/2019-01-01-12-47-57.jpg)
>面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）

$w_i$表示第 $i$ 柱子接到的雨水，那么 $h_i+w_i$ 的值由 $i$ 有左右 $min(h_j, h_K)$ 确定的。因此左右遍历数组两次，取同一位置最小值为$h_i+w_i$.
- 如果从左边考虑，每个柱子的水位应该如下图虚线所示：
![](./_image/2019-01-01-13-05-09.jpg)
- 如果从右边考虑，每个柱子的水位如下图虚线所示:

![](./_image/2019-01-01-13-07-01.jpg)

```go
func trap(height []int) int {
    if len(height) == 0 {
        return 0
    }
    leftHeight := make([]int, len(height))
    rightHeight := make([]int, len(height))
    leftHeight[0] = height[0]
    for i:= 1; i < len(height); i++ {
        if height[i] > leftHeight[i-1] {
            leftHeight[i] = height[i]
        }else{
            leftHeight[i] = leftHeight[i-1]
        }
    }
    rightHeight[len(height) - 1] = height[len(height) -1]
    for j:= len(height) -2 ; j>=0; j-- {
        if height[j] > rightHeight[j+1] {
            rightHeight[j] = height[j]
        }else{
            rightHeight[j] = rightHeight[j+1]
        }
    }
    area := 0
    for i:=0; i < len(height); i++ {
        h := leftHeight[i]
        if h > rightHeight[i] {
            h = rightHeight[i]
        }
        area = area + (h - height[i])
    }
    return area
}
```
# 2 Review
[Inversion Of Control](https://martinfowler.com/bliki/InversionOfControl.html)
当你拓展一个框架的时候，`控制反转`这个概念是一个普遍的概念，而且`控制反转`也是框架的一个特色。
让我们考虑一个简单的例子，假设你编写一个程序，需要从用户那边的到信息，如果使用命令行的话，使用`ruby`的代码形式如下
```ruby
puts 'What is your name?'
name = gets
process_name(name)
puts 'What is your quest?'
quest = gets
process_quest(quest)
```
在上述交互中，我的代码是控制整个逻辑：*它决定了什么时候问问题，什么时候读取响应，什么时候处理结果。*
但是如果我使用用户界面（windows）来完成任务，将会这样完成任务：
```ruby
require 'tk'
root = TkRoot.new()
name_label = TkLabel.new() {text "What is Your Name?"}
name_label.pack
name = TkEntry.new(root).pack
name.bind("FocusOut") {process_name(name)}
quest_label = TkLabel.new() {text "What is Your Quest?"}
quest_label.pack
quest = TkEntry.new(root).pack
quest.bind("FocusOut") {process_quest(quest)}
Tk.mainloop()
```
在这个程序中，控制流有着巨大的不同，尤其是`process_name`和`process_quest`两个方法被调用的时候。在命令行中，我控制着什么时候方法被调用，但是在window窗口中并没有，而是将控制权交给了窗口系统（也就是`Tk.mainloop()` 命令)，它决定了什么时候调用我绑定的方法。控制权被反转了，它来调用我的方法而不是我来调用框架方法。这就现象就是`控制反转`，也就是常见的`好莱坞原则`（别来调用我们，我们会调用你）
>One important characteristic of a framework is that the methods defined by the user to tailor the framework will often be called from within the framework itself, rather than from the user's application code. The framework often plays the role of the main program in coordinating and sequencing application activity. This inversion of control gives frameworks the power to serve as extensible skeletons. The methods supplied by the user tailor the generic algorithms defined in the framework for a particular application.
-- Ralph Johnson and Brian Foote

`控制反转`也是框架和库不同之处，库是一系列可以调用函数的集合，它们通常被有组织到一个类，每一次调用结束后都会将控制权返回客户端。
框架通常包含了抽象设计，内置了很多行为，为了使用它们，通常需要将你的行为插入到框架中，通常采用子类或者插件的方式，这样框架就可以调用你自定义的行为。
有很多方法可以用来调用你的代码，在上述`ruby`代码中例子中，我们通过文本框中的输入作为`lambda`表达的式参数调用绑定的方法。无论什么时候文本框检测到事件，它就能用闭包的方式调用方法。闭包非常方便，但是很多语言并不支持闭包。
另外一种方式就是很多框架定义了很多事件，客户端用来订阅这些事件。`.NET`就是一个例子，它包含了用户可以为每一个`widget`定义事件的功能，你可以使用委托来使用这些事件。
上述的方法对于简单的非常有用，但是有时候你想要在一个拓展中合并几个方法的调用，在这种情况下，框架可以定义一些接口，那么客户端代码就可以实现这些接口，来完成特定的调用。
`EJB`就是这种控制反转方式，当你开发一个会话 `bean` ，你可以在`EJB`容器调用的不同的声明周期点实现自己的方法，比如接口定义了`ejbRemove, ejbPassivate `(保存到第二级存储)， ` ejbActivate`(恢复被动状态)，当这些方法被调用的时候，并没有获取控制权，仅仅是容器调用我们，我们没有调用它们。

`控制反转`也有一个复杂的例子，但是使用场景非常简单。模板方法`template method `就是一个简单的例子，超类定义好整个控制流，子类用来它们重载的方法或者抽象方法，所以比如`JUnit`等框架代码定义了`setUp`和`tearDown`等方法来创建和销毁，当它调用的时候，你的代码做出反应，再一次控制流被反转。
但是随着`IOC`容器的流行，`控制反转`的变得模糊起来，一些人混淆了这个基础的概念。因为它的名字就让人混淆，因为`IOC`容器就是将`EJB`作为竞争对手。
# 3 Tips
在服务端开发中，仅仅单元测试不够的，还需要集成测试，也就是实际过程中开发。
# 4 Share
