---
date: 2019-01-14
status: public
tags: ARTS
title: ARTS(14)
---

# 1 Algorithm
**第k个排列**
>给出集合 [1,2,3,…,n]，其所有元素共有 n! 种排列。
按大小顺序列出所有排列情况，并一一标记，当 n = 3 时, 所有排列如下：
"123"
"132"
"213"
"231"
"312"
"321"
给定 n 和 k，返回第 k 个排列。
说明：
给定 n 的范围是 [1, 9]。
给定 k 的范围是[1,  n!]。

排列的规则如下图所示，挨个枚举所有可能的组合，但是要求排序的结果，所以没有必要输出全部组合。
![](./_image/2019-01-17-19-31-50.jpg)
对于`[1, 2, 3,...,n]`组合，如果确定第一位为`1`, 那么所有的排列组合将会有`!(n-1)`中，使用`k-!(n-1)`即可完成`2`第一位的排序。
```go
func getPermutation(n int, k int) string {
	m :=map[int]string{
		1:"1",
		2:"2",
		3:"3",
		4:"4",
		5:"5",
		6:"6",
		7: "7",
		8:"8",
		9:"9",
	}
	nums := make([]string, n)
	for i:=1; i<=n; i++ {
		nums[i-1] = m[i]
	}
	return permutaion(nums, k, "")
}

func permutaion(nums []string, k int, prefix string) string {
	if k == 0  {
		return prefix
	}else if len(nums) == 1 {
		return prefix + nums[0]
	}else{
		factor := factorial(len(nums) - 1)
		idx  := 0
		for k > factor {
			idx ++
			k = k - factor
		}
		newNums := make([]string, 0)
		for i:=0;  i<len(nums); i++{
			if i != idx{
				newNums = append(newNums, nums[i])
			}
		}
		return permutaion(newNums, k, prefix+nums[idx])


	}
}

func factorial(n int) int {
	if n <= 1  {
		return n
	}else{
		return n * factorial(n-1)
	}
}
```
# 2 Review
[Inverson of Control Container and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html) (Part II)
## 2.7 使用`Service Locator`
使用依赖注入的最大好处是它移除`MoiveLister`类必须要求具体的`MovieFinder`实现，它允许我将这个`lister`给我的朋友们，允许他们在自己的环境中以插件形式完成实现。注入并不是打破依赖的唯一方式，其他的的方法有使用`service locator`。
在`service locator`背后的主要思想是拥有个对象，它知道这个应用程序所需的全部服务，所以这个`service locator`拥有一个方法，它能够将在需要的时候返回一个`movie finder`，当然，它也是仅仅转移的负担，我们仍然需要将`locator`放置在`lister`对象中，导致依赖关系如下：

![](./_image/2019-01-14-18-45-39.png)
在这个例子中，我将使用`ServiceLocator`作为单个的注册器，`Lister`可以在初始化的时候，使用它来获得一个`finder`。
```java
class MovieLister...

  MovieFinder finder = ServiceLocator.movieFinder();
class ServiceLocator...

  public static MovieFinder movieFinder() {
      return soleInstance.movieFinder;
  }
  private static ServiceLocator soleInstance;
  private MovieFinder movieFinder;
```
使用注入的方法，我们需要配置`service locator`，在这里我将用代码进行配置，但是使用读取特定的配置文件的机制也是简单的。
```java
class Tester...

  private void configure() {
      ServiceLocator.load(new ServiceLocator(new ColonMovieFinder("movies1.txt")));
  }
class ServiceLocator...

  public static void load(ServiceLocator arg) {
      soleInstance = arg;
  }

  public ServiceLocator(MovieFinder movieFinder) {
      this.movieFinder = movieFinder;
  }
```
下面是代码
```java
class Tester...

  public void testSimple() {
      configure();
      MovieLister lister = new MovieLister();
      Movie[] movies = lister.moviesDirectedBy("Sergio Leone");
      assertEquals("Once Upon a Time in the West", movies[0].getTitle());
  }
```
我经常听到这样的抱怨，说这类`service locator`是一件不好的事情，因为由于你不能替换实现所以不可测试。当然是你可以设计得很差，导致遇到这个麻烦。在这个例子中，`service locator`实例就是拥有个简单数据，所以我能够创建一个`locator`来测试我的服务。
对于复杂的`locator`, 我可以使用`service ocator`的子类，将这个子类穿点到注册类变量。我可以将这个实例的静态方法来直接访问实例变量。我可以通过线程制定存储来提供一个线程唯一的`locator`。以上所做的都可以完成而不用改变`service locator`的客户端。
其他一种方式认为这个`service locator`是一个注册器而不是一个单例。一个单例仅仅提供简单的方式来实现注册，但是现实的决策非常容易改变。
## 2.8 为`Locator`提供隔离接口
上面的简单的实现方法有个问题就是`MovieLister`依赖整个`service locator`类，尽然它只使用了一个服务。我们可以通过`role interface`来降低它。通过这种方法，`lister`可以声明一些它需要的接口而不是全部。
在这个情况下，`lister`的提供者也将会提供它需要获得`finder`接口全部`locator`中的接口。
```java
public interface MovieFinderLocator {
    public MovieFinder movieFinder();
```
然后这个`locator`需要实现这个接口来提供访问`finder`的接口
```java
MovieFinderLocator locator = ServiceLocator.locator();
MovieFinder finder = locator.movieFinder();
public static ServiceLocator locator() {
     return soleInstance;
 }
 public MovieFinder movieFinder() {
     return movieFinder;
 }
 private static ServiceLocator soleInstance;
 private MovieFinder movieFinder;
```
你将会注意到，由于我们想要使用接口，我们将不能通过静态方法来获得服务。我们需要使用类来获得`locator`实例，然后使用它们来获得我们需要的。
## 2.9 动态的`Service Locator`
上面的例子是静态的，在`service locator`类中用拥有每一个服务需要的`service`，这并不是唯一的方式，我们可以T通过动态`service locator`来允许你隐藏任何你想要的服务。
在这个例子中，`service locator`使用的字典而不是为每个服务提供一个子弹，提供泛型来获取和加载每一个服务。
```java
class ServiceLocator...

  private static ServiceLocator soleInstance;
  public static void load(ServiceLocator arg) {
      soleInstance = arg;
  }
  private Map services = new HashMap();
  public static Object getService(String key){
      return soleInstance.services.get(key);
  }
  public void loadService (String key, Object service) {
      services.put(key, service);
  }
```
配置只需要用正确的`key`来加载服务
```java
class Tester...
  private void configure() {
      ServiceLocator locator = new ServiceLocator();
      locator.loadService("MovieFinder", new ColonMovieFinder("movies1.txt"));
      ServiceLocator.load(locator);
  }
```
使用服务也是用同样的`key`
```java
class MovieLister...
  MovieFinder finder = (MovieFinder) ServiceLocator.getService("MovieFinder");
```
但是从总体来看，我不喜欢这种方法，虽然它看上去很灵活，但是不够清晰具体。我能够找到服务的唯一方法就是通过字符串的`key`。但是我更喜欢使用明确的方法，因为这样更加容易的找到这个接口在哪里定义的。
## 2.10 `Avalon`同时使用了`locator`和注入
依赖注入和`service locator`并不是相互排斥的概念，`Avalon`框架就是很好的同时使用它们的例子。`Avalon`使用`service locator`，但是也使用注入来告诉组件去哪里发现`locator`。
`Berin Lorisch`给我发了一运行`Avalon`的简单例子
```java
public class MyMovieLister implements MovieLister, Serviceable {
    private MovieFinder finder;
 public void service( ServiceManager manager ) throws ServiceException {
        finder = (MovieFinder)manager.lookup("finder");
    } 
```
这个`service`方法就是接口注入的一个方法，允许容器来注入`service`管理器至`MyMovieLister`， 这个`service`管理器是`service locator`的例子。 在这个例子中`lister`并没有在字段中存储`manager`，而是立即使用它来查找这个`finder`，这个的确存储。
## 2.11 如何选择
到现在为止，我一直在阐述自己对这两个模式（Dependency Injection模式和ServiceLocator模式）以及它们的变化形式的看法。现在，我要开始讨论他们的优点和缺点，以便指出它们各自适用的场景。

- Service Locator vs. Dependency Injection
首先，我们面临Service Locator和Dependency Injection之间的选择。应该注意，尽管我们前面那个简单的例子不足以表现出来，实际上这两个模式都提供了基本的解耦合能力。无论使用哪个模式，应用程序代码都不依赖于服务接口的具体实现。两者之间最重要的区别在于：具体实现以什么方式提供给应用程序代码。使用Service Locator模式时，应用程序代码直接向服务定位器发送一个消息，明确要求服务的实现；使用Dependency Injection模式时，应用程序代码不发出显式的请求，服务的实现自然会出现在应用程序代码中，这也就是所谓控制反转。

控制反转是框架的共同特征，但它也要求你付出一定的代价：它会增加理解的难度，并且给调试带来一定的困难。所以，整体来说，除非必要，否则我会尽量避免使用它。这并不意味着控制反转不好，只是我认为在很多时候使用一个更为直观的方案（例如Service Locator模式）会比较合适。

一个关键的区别在于：使用Service Locator模式时，服务的使用者必须依赖于服务定位器。定位器可以隐藏使用者对服务具体实现的依赖，但你必须首先看到定位器本身。所以，问题的答案就很明朗了：选择Service Locator还是Dependency Injection，取决于对定位器的依赖是否会给你带来麻烦。

Dependency Injection模式可以帮助你看清组件之间的依赖关系：你只需观察依赖注入的机制（例如构造函数），就可以掌握整个依赖关系。而使用Service Locator模式时，你就必须在源代码中到处搜索对服务定位器的调用。具备全文检索能力的IDE可以略微简化这一工作，但还是不如直接观察构造函数或者设值方法来得轻松。

这个选择主要取决于服务使用者的性质。如果你的应用程序中有很多不同的类要使用一个服务，那么应用程序代码对服务定位器的依赖就不是什么大问题。在前面的例子中，我要把MovieLister类交给朋友去用，这种情况下使用服务定位器就很好：我的朋友们只需要对定位器做一点配置（通过配置文件或者某些配置性的代码），使其提供合适的服务实现就可以了。在这种情况下，我看不出Dependency Injection模式提供的控制反转有什么吸引人的地方。但是，如果把MovieLister 看作一个组件，要将它提供给别人写的应用程序去使用，情况就不同了。在这种时候，我无法预测使用者会使用什么样的服务定位器API，每个使用者都可能有自己的服务定位器，而且彼此之间无法兼容。一种解决办法是为每项服务提供单独的接口，使用者可以编写一个适配器，让我的接口与他们的服务定位器相配合。但即便如此，我仍然需要到第一个服务定位器中寻找我规定的接口。而且一旦用上了适配器，服务定位器所提供的简单性就被大大削弱了。

另一方面，如果使用Dependency Injection模式，组件与注入器之间不会有依赖关系，因此组件无法从注入器那里获得更多的服务，只能获得配置信息中所提供的那些。这也是Dependency Injection 模式的局限性之一。

人们倾向于使用Dependency Injection模式的一个常见理由是：它简化了测试工作。这里的关键是：出于测试的需要，你必须能够轻松地在真实的服务实现与供测试用的伪组件之间切换。但是，如果单从这个角度来考虑，Dependency Injection模式和Service Locator模式其实并没有太大区别：两者都能够很好地支持伪组件的插入。之所以很多人有Dependency Injection模式更利于测试的印象，我猜是因为他们并没有努力保证服务定位器的可替换性。这正是持续测试起作用的地方：如果你不能轻松地用一些伪组件将一个服务架起来以便测试，这就意味着你的设计出现了严重的问题。

当然，如果组件环境具有非常强的侵略性（就像EJB框架那样），测试的问题会更加严重。我的观点是：应该尽量减少这类框架对应用程序代码的影响，特别是不要做任何可能使编辑-执行的循环变慢的事情。用插件（plugin）机制取代重量级组件会对测试过程有很大帮助，这正是测试驱动开发（Test Driven Development，TDD）之类实践的关键所在。

所以，主要的问题在于：代码的作者是否希望自己编写的组件能够脱离自己的控制、被使用在另一个应用程序中。如果答案是肯定的，那么他就不能对服务定位器做任何假设——哪怕最小的假设也会给使用者带来麻烦。

- 构造函数注入 vs. 设值方法注入
在组合服务时，你总得遵循一定的约定，才可能将所有东西拼装起来。依赖注入的优点主要在于：它只需要非常简单的约定——至少对于构造函数注入和设值方法注入来说是这样。相比于这两者，接口注入的侵略性要强得多，比起Service Locator模式的优势也不那么明显。所以，如果你想要提供一个组件给多个使用者，构造函数注入和设值方法注入看起来很有吸引力。你不必在组件中加入什么希奇古怪的东西，注入器可以相当轻松地把所有东西配置起来。

设值函数注入和构造函数注入之间的选择相当有趣，因为它折射出面向对象编程的一些更普遍的问题：应该在哪里填充对象的字段，构造函数还是设值方法？

一直以来，我首选的做法是尽量在构造阶段就创建完整、合法的对象——也就是说，在构造函数中填充对象字段。这样做的好处可以追溯到Kent Beck在Smalltalk Best Practice Patterns一书中介绍的两个模式：Constructor Method和Constructor Parameter Method。带有参数的构造函数可以明确地告诉你如何创建一个合法的对象。如果创建合法对象的方式不止一种，你还可以提供多个构造函数，以说明不同的组合方式。

构造函数初始化的另一个好处是：你可以隐藏任何不可变的字段——只要不为它提供设值方法就行了。我认为这很重要：如果某个字段是不应该被改变的，没有针对该字段的设值方法就很清楚地说明了这一点。如果你通过设值方法完成初始化，暴露出来的设值方法很可能成为你心头永远的痛。（实际上，在这种时候我更愿意回避通常的设值方法约定，而是使用诸如initFoo之类的方法名，以表明该方法只应该在对象创建之初调用。）

不过，世事总有例外。如果参数太多，构造函数会显得凌乱不堪，特别是对于不支持关键字参数的语言更是如此。的确，如果构造函数参数列表太长，通常标志着对象太过繁忙，理应将其拆分成几个对象，但有些时候也确实需要那么多的参数。如果有不止一种的方式可以构造一个合法的对象，也很难通过构造函数描述这一信息，因为构造函数之间只能通过参数的个数和类型加以区分。这就是Factory Method模式适用的场合了，工厂方法可以借助多个私有构造函数和设值方法的组合来完成自己的任务。经典Factory Method模式的问题在于：它们往往以静态方法的形式出现，你无法在接口中声明它们。你可以创建一个工厂类，但那又变成另一个服务实体了。工厂服务是一种不错的技巧，但你仍然需要以某种方式实例化这个工厂对象，问题仍然没有解决。

如果要传入的参数是像字符串这样的简单类型，构造函数注入也会带来一些麻烦。使用设值方法注入时，你可以在每个设值方法的名字中说明参数的用途；而使用构造函数注入时，你只能靠参数的位置来决定每个参数的作用，而记住参数的正确位置显然要困难得多。

如果对象有多个构造函数，对象之间又存在继承关系，事情就会变得特别讨厌。为了让所有东西都正确地初始化，你必须将对子类构造函数的调用转发给超类的构造函数，然后处理自己的参数。这可能造成构造函数规模的进一步膨胀。

尽管有这些缺陷，但我仍然建议你首先考虑构造函数注入。不过，一旦前面提到的问题真的成了问题，你就应该准备转为使用设值方法注入。

在将Dependecy Injection 模式作为框架的核心部分的几支团队之间，构造函数注入还是设值方法注入引发了很多的争论。不过，现在看来，开发这些框架的大多数人都已经意识到：不管更喜欢哪种注入机制，同时为两者提供支持都是有必要的。

- 代码配置 vs. 配置文件
另一个问题相对独立，但也经常与其他问题牵涉在一起：如何配置服务的组装，通过配置文件还是直接编码组装？对于大多数需要在多处部署的应用程序来说，一个单独的配置文件会更合适。配置文件几乎都是XML 文件，XML 也的确很适合这一用途。不过，有些时候直接在程序代码中实现装配会更简单。譬如一个简单的应用程序，也没有很多部署上的变化，这时用几句代码来配置就比XML 文件要清晰得多。

与之相对的，有时应用程序的组装非常复杂，涉及大量的条件步骤。一旦编程语言中的配置逻辑开始变得复杂，你就应该用一种合适的语言来描述配置信息，使程序逻辑变得更清晰。然后，你可以编写一个构造器（builder）类来完成装配工作。如果使用构造器的情景不止一种，你可以提供多个构造器类，然后通过一个简单的配置文件在它们之间选择。

我常常发现，人们太急于定义配置文件。编程语言通常会提供简捷而强大的配置管理机制，现代编程语言也可以将程序编译成小的模块，并将其插入大型系统中。如果编译过程会很费力，脚本语言也可以在这方面提供帮助。通常认为，配置文件不应该用编程语言来编写，因为它们需要能够被不懂编程的系统管理人员编辑。但是，这种情况出现的几率有多大呢？我们真的希望不懂编程的系统管理人员来改变一个复杂的服务器端应用程序的事务隔离等级吗？只有在非常简单的时候，非编程语言的配置文件才有最好的效果。如果配置信息开始变得复杂，就应该考虑选择一种合适的编程语言来编写配置文件。

在Java 世界里，我们听到了来自配置文件的不和谐音——每个组件都有它自己的配置文件，而且格式还各不相同。如果你要使用一打这样的组件，你就得维护一打的配置文件，那会很快让你烦死。

在这里，我的建议是：始终提供一种标准的配置方式，使程序员能够通过同一个编程接口轻松地完成配置工作。至于其他的配置文件，仅仅把它们当作一种可选的功能。借助这个编程接口，开发者可以轻松地管理配置文件。如果你编写了一个组件，则可以由组件的使用者来选择如何管理配置信息：使用你的编程接口、直接操作配置文件格式，或者定义他们自己的配置文件格式，并将其与你的编程接口相结合。

分离配置与使用
所有这一切的关键在于：服务的配置应该与使用分开。实际上，这是一个基本的设计原则——分离接口与实现。在面向对象程序里，我们在一个地方用条件逻辑来决定具体实例化哪一个类，以后的条件分支都由多态来实现，而不是继续重复前面的条件逻辑，这就是分离接口与实现的原则。
如果对于一段代码而言，接口与实现的分离还只是有用的话，那么当你需要使用外部元素（例如组件和服务）时，它就是生死攸关的大事。这里的第一个问题是：你是否希望将选择具体实现类的决策推迟到部署阶段。如果是，那么你需要使用插入技术。使用了插入技术之后，插件的装配原则上是与应用程序的其余部分分开的，这样你就可以轻松地针对不同的部署替换不同的配置。这种配置机制可以通过服务定位器来实现（Service Locator模式），也可以借助依赖注入直接完成（Dependency Injection 模式）。

# 3 Tips
 ## 3.1  emacs 替换
  `M-x replace-string <RET> string <RET> newstring <RET>` 
 ## 3.2 git 在push之后压缩commit
Squash commits locally with
`git rebase -i origin/master~4 master`
and then force push with
`git push origin +master` or `git push --force origin master`
## 3.3 git merge 策略选择
# 4 Share