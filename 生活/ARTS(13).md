---
date: 2019-01-07
status: public
tags: ARTS
title: ARTS(13)
---

# 1 Algorithm
>给定一个未排序的整数数组，找出最长连续序列的长度。
要求算法的时间复杂度为 O(n)。
> 输入: [100, 4, 200, 1, 3, 2]
输出: 4
解释: 最长连续序列是 [1, 2, 3, 4]。它的长度为 4。

既然保证算法的时间复杂度为$O(n)$，那么基本上排除先排序再处理的方式。对于连续队列，只需要找到其中的一个元素，不停的自增或者自减查找该值是否在数组中出现过，那么就需啊借助`map`数据结构保存查找信息。为了避免一个序列中的元素重复查找，一旦查找完毕该元素，则将其删除，同样使用链表保存信息，增加的删除的时间时间复杂度为$O(1)$。
```go
type Element struct {
    value int
    prev *Element
    next *Element
}

func newElement(val int) *Element{
    return &Element { value: val}
}

func (e *Element) Drop() {
    e.prev.next = e.next
    e.prev.next.prev = e.prev
}

func (e *Element) InsertBack (other *Element) {
    //
    other.next = e.next
    e.next.prev = other
    //
    other.prev = e
    e.next = other
}

func (e *Element) InsertForward(other *Element){
    //
    other.prev = e.prev
    e.prev.next = other
    //
    other.next = e
    e.prev = other
}

type List struct {
    size int
    head *Element
    tail *Element
}

func NewList() *List {
    l := &List{
        size: 0,
        head: newElement(0),
        tail: newElement(0),
    }
    l.head.next = l.tail
    l.tail.prev = l.head
    return l
}
func (l *List) Size() int {
    return l.size 
}
func (l *List) Empty() bool {
    return l.Size() <= 0
}
func (l *List) InsertFront(e *Element) {
    l.size++
    l.head.InsertBack(e)
}
func (l *List) InsertBack(e *Element){
    l.size++
    l.tail.InsertForward(e)
}

func (l *List) First() *Element{
    return l.head.next
}
func (l *List) Remove(e *Element) {
    l.size--
    e.Drop()
}

func longestConsecutive(nums []int) int {
    l := NewList()
    m := make(map[int]*Element, len(nums))
    for _, num := range nums {
        if _, ok := m[num]; !ok {
            e := newElement(num)
            l.InsertBack(e)
            m[num] = e
        }
    }
    longest := 0
    for !l.Empty() {
        i := 1 
        e := l.First()
        val := e.value
        //delete(m, val)
        l.Remove(e)
        // up 
        for {
            val++
            if up, ok:=m[val]; ok {
                i++
                l.Remove(up)
                //delete(m, val)
            }else{
                break
            }
        }
        //down
        val = e.value
        for {
            val--
            if down, ok := m[val]; ok {
                i++
                l.Remove(down)
                //delete(m, val)
            }else{
                break
            }
        }
        if i > longest{
            longest = i
        } 
    }
    return longest
}
```

# 2 Reivew
[Inversion of Control Container and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html)(Part I)
**控制反转容器和依赖注入模式(上)** 
在Java世界有一个比较好玩的事在开源世界都在构建替代主流的J2EE技术，其中大部分都是对主流的J2EE技术的复杂度做出回应。一个常见的议题就是如何将不同的元素合并起来，如何将web控制器适应于不同团队的构建的数据库接口等等。有一些框架已经着手开始处理这些问题，其中有一些提供一些 通用能力在不同层组装各个成分。这些通常是指一些轻量级的容器，比如`PicoContainer`和`Spring`
在这些容器中，包含许多有趣的设计原则，这些超出了特定的容器和Java平台，在这我想开始探索这些原则，例子中我将使用Java，但是这些原则和其他OOP环境类似，尤其是`.NET`。
## 2.1 Component和Service
`组装元素`这个概念让我立马想起了概念就是`组分`和`服务`，在这两个概念上有大量相互矛盾的文章，对我来说我将超出这个概念来讨论。
我使用`compoent`是指一系列不会改变的软件，通常应用软件不会修改这些`component`的代码，尽管可以通过拓展方法来修改其行为。
`service`和`compoent`比较类似都是被外部应用程序使用，主要的不同的是我认为`component`是在本地的，比如`jar`文件，`dll`等。而`Service`通常是在远处以供远处接口，即可以是同步的，也可以是异步的。
在这边文章中，我将大部分使用`service`这个概念，但是这些都可以应用到本地`component`中。事实上你需要一些本地的`compoent`来更容易的访问远处的`service`。但是写`component or service`太累了，使用`service`显得更加时髦一点。

## 2.2 简单的例子
为了讨论更加具体点，我将使用一个例子来说明。就像其他一样，这是非常简单的例子。虽然简单，但是足够让你明白我讨论的内容。
在这个例子中，我将写一个`component`用来返回特定导演的电影列表，有一个简单的方法来实现
```java
//class MovieLister
public Movie[] moviesDirectedBy(String arg) {
      List allMovies = finder.findAll();
      for (Iterator it = allMovies.iterator(); it.hasNext();) {
          Movie movie = (Movie) it.next();
          if (!movie.getDirector().equals(arg)) it.remove();
      }
      return (Movie[]) allMovies.toArray(new Movie[allMovies.size()]);
  }
```
这个实现过程非常简单，它请求一个`finder`对象，然后返回所有的电影。然后它迭代它的列表中的每一个电影，然后返回每一个电影的导演等于给定的导演。这个代码非常简单我也不会去修改它，因为它是我们这篇文章的脚手架。
这篇文章的重点是`finder`对象，后者说我们该如何将`lister`对象和特定的`finder`对象连接起来，它非常有趣的原因是我想让我的`moviesDirectedBy`方法能够完全独立于电影是如何存储的。所以所有的方法都知道`finder`对象，而且所有的`finder`对象都知道如何响应`findAll`方法。我可以将这个方法单独提取出来定义为一个接口
```java
public interface MovieFinder {
    List findAll();
}
```
现在所有的都解耦了，与此同时我需要写出具体的类来处理电影，在这里我将在构造函数里添加如下代码
```java
// class MoveLister...
private MovieFinder finder;
  public MovieLister() {
    finder = new ColonDelimitedMovieFinder("movies1.txt");
  }
```
实现的具体类名称是基于如下的事实，我将从以冒号间隔的文本文件中获取电影列表。
现在如果我仅仅自己使用这个类，一切将如期工作。但是如果我的朋友非常喜欢我这件工作，希望将这个程序拷贝回去。但是如果他将电影列表保存在有冒号隔开的文本文件中，一切都正常工作。但是如果他将其放在不同形式的存储中，比如关系型数据库中，`XML`数据库，或者一个`web`服务中。在这种情况下，我们需要不同的类来获取数据。因为我已经定义了`MovieFinder`接口，它也不会改变我们的`moviesDirectedBy`方法。但是我忍让需要其他方法能够将右边的`finder`实现给替换掉。

![](./_image/2019-01-06-15-56-39.png)
上图显示了这种情况下的依赖关系，`MovieLister`类依赖于`MoiveFinder`接口和具体的实现类。我们更喜欢它仅仅依赖接口，但是我们该如何取得一个工作的实例呢？
在我的`P of EAA`中，我将其描述为一个插件，`finder`接口实现类不是在编译的时候连接过去，因为我不知道我们的朋友将会如何使用它。而是我希望我的`lister`能够和任何实现方式工作，实现的方式将会在以后某一个时刻作为插件使用。问题就是我该如何是我的`lister`类与具体的实现类无关，但是仍然用具体的实例来工作。
来到现实系统中，我们可能拥有很多`service`和`component`，在每一种情况下，我们都可以抽象出我们使用这些`component`，而是使用接口。如果我们想要以不同的方式部署我们的系统，我们需要插件与不同的服务来交互，以便达到不同的部署方式。
所以核心问题是我们该如何组装这些插件到一个应用程序中呢？这也是现在新的容器所面临的问题难题，通常他们使用控制反转。

## 2.3 控制反转
当这些容器在讨论它们是如何有用因为他们实现了`控制反转`，我就变得非常疑惑了。因为控制反转是框架的基本特征，所以说轻量级的容器很棒因为它使用了控制反转就像说我的汽车特别棒因为它拥有轮子。
那么问题来了，究竟是哪一方面是被反转了？当我第一次听说控制反转，主要的就是用户界面。早期的用户界面通常是由应用程序控制的，你将会有一系列命令：`输入名字`，`输入地址`等等。你的应用程序将会输出提示并且选择各自的回应。使用图形界面，UI框架将会包含整个循环，你的程序为屏幕上大量的界面提供事件注册函数。这样整个程序的控制权被反转，从你的的手中转移到框架中。
对容器来说，控制反转就是如何查询插件的实现。在我最简单的例子中，`lister`查询的`finder`接口就直接内部实例化了，它阻止了将`finder`变成一个插件。这种容器用确保每一个插件遵循一定的规范来允许每一个独立的组件能够被注入到`lister`中去。
我想我们应该需要为这个模式起一个具体的名字，控制反转是一个泛泛的概念，所以人们疑惑，从IOC不同种拥护者来看，我们应该叫它`依赖注入`。
我将开始讨论不同形式的依赖注入，但是我需要指出的是不仅仅是只有一种方式将依赖从应用程序类转移到插件实现；其他你可以使用的模式有服务发现，在解释完依赖注入后我会讨论这个话题。
## 2.4 依赖注入的形式
依赖注入的基本形式就是拥有隔离的对象，一个组件，它拥有`lister`对象所需的接口的全部字段，那么现在依赖图如下所示：

![](./_image/2019-01-08-22-19-25.png)
主要有三种形式的依赖注入，我为他们起的名字有`构造注入`，`属性注入`和`接口注入`。如果你看过之前讨论的关于控制反转的内容，你可能听说过`类型1控制反转`，`类型2控制反转`和`类型3控制反转`，我觉得数字命名非常难记得，这也是为什么在这里将使用名字命名。
## 2.5 `PicoContainer`构造注入
我将以轻量级的容器`PicoContainer`作为开始来展示注入是如何完成的，至于为什么使用这个因为我在`ThoughtWork`工作的好几个同事活跃于`PicoContainer`容器的开发中。
`PicoContainer`使用构造函数来决定如何在`lister`类中实现注入`finder`接口的实现，在这个例子中，`lister`类需要包含每一个构造函数，它包含了所有需要注入的元素。
```java
//class MovieLister...
    public MovieLister(MoiveFinder finder){
        this.finder = finder;
    }
```
`finder`对象本身将由`pico container`来管理，比如我们有一有个文本文件名被容器锁注入
```java
//class ColonMovieFinder
public ColonMovieFinder(String fileName){
    this.filename = filename;
}
```
`pico container`需要被告知哪一个具体的实现类对应的各自的接口，还有哪一个字符串被注入到相应的接口
```java
private MutablePicoContainer configureContainer() {
    MutablePicoContainer pico = new DefaultPicoContainer();
    Parameter[] finderParams =  {new ConstantParameter("movies1.txt")};
    pico.registerComponentImplementation(MovieFinder.class, ColonMovieFinder.class, finderParams);
    pico.registerComponentImplementation(MovieLister.class);
    return pico;
}
```
上面的配置代码可以在用在不同的类中，比如每一个使用我的`lister`的朋友它们可以为自己的编写这个启动类，当然通常的做法是用一个单独的配置文件来保存这种配置信息。你可写一个类来读取这个配置文件然后正确的启动容器。虽然目前`PicoContainer`并不包含这个功能，但是与之类似的`NanoContainer`可以提供特定的封装，然你可以使用`XML`配置文件。这样的`nano container`将会解析XML，然后配置一下包含的`pico container`。这种项目的哲学就是将配置文件的格式和包含的机制隔离开来。
使用容器，你将会写下如下的代码
```java
public void testWithPico() {
    MutablePicoContainer pico = configureContainer();
    MovieLister lister = (MovieLister) pico.getComponentInstance(MovieLister.class);
    Movie[] movies = lister.moviesDirectedBy("Sergio Leone");
    assertEquals("Once Upon a Time in the West", movies[0].getTitle());
}
```
尽管在上述的例子中，我使用了构造注入，`PicoContainer`还支持属性注入，但目前开发者还是偏向于构造注入。
## 2.6 Spring属性注入
`Spring`框架被广泛使用在企业级开发应用中，它包含了事务抽象层，持久化框架，web应用程序和JDBC，像`PicoContainer`它既支持构造注入也支持属性注入，但是`Spring`开发者更喜欢属性注入，所以使用它作为例子来讲解。
为了让我的`Lister`能够接受注入，我为这个服务定义一个设值的方法
```java
// class MovieLister...
private MovieFinder finder;
public void setFinder(MovieFinder finder) {
  this.finder = finder;
}
```
同样的，定义为`filename`定义个设值方法
```java
// class ColonMovieFinder...
public void setFilename(String filename) {
      this.filename = filename;
  }
```
第三步就是建议个配置文件，`Spring`支持使用`XML`文件也支持通过代码，但是`XML`是更常用的方法
```xml
<beans>
    <bean id="MovieLister" class="spring.MovieLister">
        <property name="finder">
            <ref local="MovieFinder"/>
        </property>
    </bean>
    <bean id="MovieFinder" class="spring.ColonMovieFinder">
        <property name="filename">
            <value>movies1.txt</value>
        </property>
    </bean>
</beans>
```
测试看上去是这样的
```java
public void testWithSpring() throws Exception {
    ApplicationContext ctx = new FileSystemXmlApplicationContext("spring.xml");
    MovieLister lister = (MovieLister) ctx.getBean("MovieLister");
    Movie[] movies = lister.moviesDirectedBy("Sergio Leone");
    assertEquals("Once Upon a Time in the West", movies[0].getTitle());
}
```
## 2.6 接口注入
第三种注入技术是定义和使用接口来注入。`Avalon`是使用这种技术的一个代表。后面我会介绍他，但是在这个例子中，我将使用一些简单的代码。
使用这种技术，我开始先定义一些接口来完成注入途径，下面的接口就是用注入`movie finder`到一个具体的类中
```java
public interface InjectFinder {
    void injectFinder(MovieFinder finder);
}
```
这个接口将会被任何`MovieFinder`具体类定义，它也被任何需要`finder`对象的类所实现，比如`lister`
```java
// class MovieLister implements InjectFinder
  public void injectFinder(MovieFinder finder) {
      this.finder = finder;
  }
```
我将用相似的方法来注入`filename`来完成具体的`finder`实现
```java
public interface InjectFinderFilename {
    void injectFilename (String filename);
}
```
`ColonMovieFinder`类将会实现`MovieFinder`和`InjectFinderFilename`
```java 
 public void injectFilename(String filename) {
      this.filename = filename;
  }
```
接下来，和之前一样我需要配置代码来讲这个实现捆绑在一起，从简单期间，我将用如下代码实现
```java
//class Tester...
  private Container container;
   private void configureContainer() {
     container = new Container();
     registerComponents();
     registerInjectors();
     container.start();
  }
```
整个配置过程有两步，通过查找`key`来注册成分，这个和起来例子相同
```java
// class Tester...
private void registerComponents() {
    container.registerComponent("MovieLister", MovieLister.class);
    container.registerComponent("MovieFinder", ColonMovieFinder.class);
  }
```
新的一步是注册注入器，它将会注入依赖的成分。每一个注入的接口需要代码来注入依赖的UI想。在这里通过注册容器的中对象。每一个注册对象实现了注入器的接口
```java
class Tester...

  private void registerInjectors() {
    container.registerInjector(InjectFinder.class, container.lookup("MovieFinder"));
    container.registerInjector(InjectFinderFilename.class, new FinderFilenameInjector());
  }
public interface Injector {
  public void inject(Object target);
}
```
当为这个容器编写的依赖，这个需要所有的祖册必须要实现注入器本省。在这里就是`movie finder`,从泛型角度，就是字符串。我将使用内部类来事项配置代码
```java 
// class ColonMovieFinder implements Injector...
  public void inject(Object target) {
    ((InjectFinder) target).injectFinder(this);        
  }
// class Tester...
  public static class FinderFilenameInjector implements Injector {
    public void inject(Object target) {
      ((InjectFinderFilename)target).injectFilename("movies1.txt");      
    }
    }
```
测试代码如下
```java
// class Tester…
public void testIface() {
    configureContainer();
    MovieLister lister = (MovieLister)container.lookup("MovieLister");
    Movie[] movies = lister.moviesDirectedBy("Sergio Leone");
    assertEquals("Once Upon a Time in the West", movies[0].getTitle());
  }
```
容器使用声明的注入接口来搞清依赖，注入器来注入正确的依赖。
(未完待续)
# 3 Tips
1. 信号量也是进程之间通信的方式；
# 4 Share
`Rob Pike`编程原则
1. 你没有办法预测每个程序的运行时间，瓶颈会出现在出乎意料的地方，所以在分析瓶颈原因之前，先不要盲目猜测。
2.  测试（measure）。在测试之前不要优化程序，即使在测试之后也要慎重，除非一部分代码占据绝对比重的运行时间。
3. 花哨的算法在 n 比较小时效率通常比较糟糕，而 n 通常是比较小的，并且这些算法有一个很大的常数。除非你确定 n 在变大，否则不要用花哨的算法。（即便 n 不变大，也要先遵循第 2 个原则。）
4. 相对于朴素的算法来说，花哨的算法更容易出现Bug，更难调试。尽量使用朴素的算法和数据结构。
5. 数据占主导地位（Data dominates）。如果你选择了正确的数据结构，并且已把事情组织好，那么算法的效率显而易见。编程的核心是数据结构是，不是算法。