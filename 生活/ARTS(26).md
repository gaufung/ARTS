---
date: 2019-04-15
status: public
tags: ARTS
title: ARTS(26)
---

# 1 Algorithm
> 在一个数组中，只包含 `0, 1, 2` 三个数据，在 `O(n)` 时间复杂度和 `O(1)` 空间复杂度的情况下，将数组排序。

我们知道基于比较的数组排序算法中，时间复杂度的下限是 $nlog(n)$，而 `O(1)` 空间复杂度要求不能借助字典数据结构。 由于数组中只包含 `0, 1, 2` 三种数值，那么排序后的数组，必定形如 `[0, 0 ... 1, 1 ..., 2, 2 ... ]`，那么我们只需要遍历整个数组，将 `0` 与数组前面的元素交换，将 `2` 与数组后面的元素交换。因此我们需要三个指针:
- `pivot`: 指向正在比较的数组元素；
- `header`: 数组前面第一个非 `0` 元素；
- `tailer`:  数组后面第一个非 `2` 元素；

**算法正确性证明**
算法正确性证明要求两点：
- 单调性
- 不变性

数组中 $[0: header) = 0$ 和 $[tailer+1: length) = 2$ 这两个是不变的；$header \le pivot \le tailer$ 这个不等式也是一直成立的。
`pivot` 指针在每次迭代会出现下面的三种情况
1. `nums[pivot] == 1`:  这时候 `pivot++`
2. `nums[pivot] == 0` 这时 `nums[pivot]` 和 `nums[header]` 交换， `header++`
3. `nums[pivot] == 2` 这时候 `nums[pivot]` 和 `nums[tailer]` 交换， `tailer` 值减小（可能出现连续 `2` 值）
所以单调性和不变形都满足条件，算法正确。
```go
func sortColors(nums []int) {
	if len(nums) <= 1 {
		return
	}
	if len(nums) == 2 {
		if nums[0] > nums[1] {
			nums[0], nums[1] = nums[1], nums[0]
		}
		return
	}
	header, tailer := forward(0, nums), backward(len(nums)-1, nums)
	if header == len(nums) || tailer == -1 {
		return
	}
	pivot := header
	for pivot <= tailer {
		if nums[pivot] == 0 {
			nums[header], nums[pivot] = nums[pivot], nums[header]
			header = forward(header, nums)
			pivot = header
			continue
		}
		if nums[pivot] == 2 {
			nums[tailer], nums[pivot] = nums[pivot], nums[tailer]
			tailer = backward(tailer, nums)
			continue
		}
		if nums[pivot] == 1 {
			pivot++
		}
	}
}

func forward(header int, nums []int) int {
	if header < len(nums) && nums[header] == 0 {
		header++
	}
	return header
}

func backward(tailer int, nums []int) int {
	if tailer > -1 && nums[tailer] == 2 {
		tailer--
	}
	return tailer
}
```
# 2 Review
[All About EOF](https://latedev.wordpress.com/2012/12/04/all-about-eof/)
**关于EOF**
## 2.1 介绍
在 Reddit 和 StackOverflow 网站上很多 C/C++ 初学者都会提出最常见的问题是在编写用户交互和读取文件的时候如何处理文件末尾问题处理。我估计超过 $95\%$ 的问题都是对于文件末尾 (`end -of-file`) 概念的误解。这篇文章将会解释所有跟这个主题相关的混淆的概念，主要是针对 C/C++ 程序员，而且对于不同额操作系统有特别的说明。

## 2.2 `EOF`字符的谜团
第一个问题是很多初学者在面对 `end-of-line` 问题的时候，都会认为存在 `EOF` 字符。但是对于 `Windows` 和 `Linux` 都不存在文件结束后标记的概念。如果你在 `Windows` 上使用记事本创建一个文件，同样的用 `Vim` 在 `Linux` 上创建文件或者其他任何你喜欢的文本编辑器，你可以发现没有任何地方有特别的字符标记文件结束。`Windows` 和 `Linux` 的文件系统都知道文件内容确切的字节长度，所以没有必要用特殊的字符标记文件结束。
既然 `Windows` 和 `Linux` 都使用了 `EOF` 字符， 那么这个做法是从那沿袭过来的呢？在很久以前，有一个叫做 `CP/M` 的操作系统，它运行在 8 位处理器上，比如 `Zilog Z80` 或者 `Intel 8080` 。`CP/M` 操作系统不知道文件字节的确切长度。它只知道文件占据了磁盘多少块。这就意味着如果编辑了一个很小只包含 `hello world` 文本文件，`CP/M` 不知道这个文件共 `11` 个字节长，它只知道文件值占据了一个磁盘块，因为磁盘块大小的最小值为 `128` 字节。由于人们更想知道他们的文件有多大，而不是磁盘块占据的数量，所以文件结束标识符被提了出来。`CP/M` 重用了 `Control-Z` 标识符（值为 26， 十六进制 1A )，在 ASCII 码中也表示同样的目的。当 `CP/M` 应用程序读取了 `Control-Z` 字符，将它当做已经读取到文件的末尾。没有强制要求应用必须这么处理，处理二进制应用程序需要其他方式来确定是否到达文件的末尾，而且操作系统也没有特殊处理 `Control-Z` 字符。
当 `MS-DOS` 出现了， 兼容 `CP/M` 变得非常重要。`MS-DOS` 开始的很多应用程序都是从 `CP/M` 系统中转换过来的。由于这些应用程序没有重写，他们依然将 `Control-Z` 当做文件结束标识符。事实上，`Control-Z` 的处理方式已经内置到 `Microsoft` C 运行库中。如果一个文件已文本的模式打开，它通知 `Windows` 操作系统除了 `Control-Z` 之外其余的都不关心。这个特性在根植与微软库中，不幸的是每一个 `Windows` 应用程序都会使用。所以要记住，这个完全是 `Windows` 的问题， `Linux` 或者其他 `Unix` 都不会将 `Control-Z` (或者其他的字符) 当做文件的表示符，不管以什么形式打开文件。

## 2.3 示例代码
你可以通过微软库代码来演示这个”不幸”的功能，首先编写一个将 `Control-Z` 输出到文本中的程序。
```cpp
#include <iostream>
#include <fstream>
using namespace std;
int main(){
    ofstream ofs("myfile.txt");
    ofs << "line 1\n";
    ofs << char(26);
    ofs << "line 2\n";
}
```
如果你在 `Windows` 或者 `Linux` 上编译并且运行这个程序，就会得到一个文本文件，它在两行文本中间嵌入了一个 `Control-Z` (`ASCII` 码为 26 )。在任何平台上都没有什么特殊输出含义，你可以尝试使用控制命令来读这个文件。在 `Windows` 上
```
c:\users\neilb\home\temp>type myfile.txt
line 1
```
注意到只有第一行被展示。在 `Linux` 上
```
[neilb@ophelia temp]$ cat myfile.txt
line 1
?line 2
```
两行文本都展示出来，但是中间包含了一个奇怪的字符（在这里使用问号表示）。`cat` 命令读取这个`Control-Z` 字符并且和其他字符一样打印出来，具体得到什么字符取决于你的的终端程序。
这个看上去好像 `Windows` 操作系统知道 `Control-Z` 字符。但是这不是这样的，这个取决于特定的应用程序知道这一点。如果你的使用 `Windows` 记事本打开，你将会看到这个：
![](./_image/2019-04-17-22-35-00.jpg)
所有行都显示出来，并且中间包含了一个 `Control-Z` 字符。记事本并不知道 `Control-Z` 是文件结束的标识符。
## 2.4 文本和二进制模式
这个很难说 `type` 命令和记事本应用程序究竟有什么区别，可能 `type` 命令有一些特殊代码来检查输入的 `Control-Z` 字符。但是 `Windows` 程序员在使用 `c++ iostream` 库或者 `C stream` 库的时候有选项在打开文件的时候选择文本模式或者二进制模式，这个对于我们获取的内容有产生区别。下面是说那个 `C++` 读取文本文件的方式
```c++
#include <iostream>
#include <fstream>
#include <string>
using namespace std;

int main() {
    ifstream ifs( "myfile.txt" );
    string line;
    while( getline( ifs, line ) ) {
        cout << line << '\n';
    }
}
```
如果你在 `Windows` 平台编译和运行这个文件，你会看到 `Control-Z` 被当做文件的结束的标识符，输出如下
```
line 1
```
然而，如果你以二进制模式打开
```cpp
ifstream ifs( "myfile.txt", ios::binary );
```
输出的结果
```
line 1
?line 2
```
 `Control-Z` 只有在文本模式当做特殊处理，在二进制模式中和其他字符是一样的。注意着一点只针对 `Windows`，在 `Linux` 平台中上述的不同打开模式的程序的行为是一样的。
要注意两点：
- 如果你想让你的文件在文本模式中正确打开，请不要将包含 `Control-Z` 字符；
- 如果你必须要包含 `Control-Z` 字符，使用二进制模式打开。

## 2.5 Control-D 是什么
一些 `Linux` 用户这个时候开始疑问：“当我结束在 Shell 输入时候输入的 `Control-D` 是做什么的？它不是文件结束符吗？"。是的，它也不是文件终止符。如果你在文本符中包含 `Control-D` 字符， `Linux` 并不会特别注意它，事实上你在 `Shell` 命令中输入 `Control-D` 是给 `Shell` 进程发送了信号，让它关闭标准输入流，并没有在输入流中插入特定的符号。事实上，使用 `stty` 工具你可以修改任何字符来替换 `Control-D` 来结束文件输入流，而不是在输入流中插入任何特殊的字符。所以无论哪种方式，`Linux` 都不会将它们当做文件结束标识符。

## 2.5 C/C++ 中 `EOF` 值
让人混淆的是，在 `C/C++` 中都定义了特殊的值为 `EOF`, 在 C 标准库中，它定义在 `<stdio.h>` 中
```c
#define EOF (-1)
```
同样定义的在 C++ 的 `<cstdio>` 中。
注意到这里 `EOF` 和 `Control-Z` 没有任何关系。它的值并不是 26 实际上更不是字符而是一个整数。它被用在下面的函数返回值中：
```c
int getchar(void);
```
`getchar` 函数用来从标准输入中读取每一个字符并且如果遇到文件结束返回特殊的 `EOF` 值。文件结束标识符可以使用 `Control-Z` 字符标记，但是任何情况下 `EOF` 的值都不等于 `Control-Z` 的 ASCII 码。事实上 `getchar()` 函数返一个整数而不是字符，它将字符存储到整数中。作为对比字符和整数并不保证能够正确工作，下面是读取标准输入的常规的做法
```c
#include <stdio.h>
int main() {
    int c;
    while( (c = getchar()) != EOF ) {
        putchar( c );
    }
}
```
## 2.6 `eof()` 和 `feof()` 函数
另一个层面的疑惑是 `C/C++` 提供了检查输入流状态的函数，大部分编程初学者被这些函数搞糊涂了，是时候我们弄清楚到底怎么使用这些函数。
> `eof()` 和 `feof()` 函数都是用来检查输入流状态，判断是否满足文件末尾的条件，这个条件只有在尝试读的时候满足。如果你没有进行读操作而开始调用这个函数，那么你的代码就错了。永远不要循环 `eof` 函数。

为了表达清楚，我们编写读文件的程序，它为文件每一行添加行号。为了简化问题，我们使用固定的文件名并且跳过错误检查。大部分初学者会这样写代码
```cpp
#include <iostream>
#include <fstream>
#include <string>
using namespace std;
int main() {
    ifstream ifs( "afile.txt" );
    int n = 0;
    while( ! ifs.eof() ) {
        string line;
        getline( ifs, line );
        cout << ++n << " " << line << '\n';
    }
}
```
这个看上去好像没有问题，但是注意我们的之前你的建议：
> 如果你没有进行读操作而开始调用这个函数，那么你的代码就错了。

在上述的例子中我们的确在没有读之前调用了 `eof()` 函数。为了判断到底哪里错了，考虑一下 `afile.txt` 为空的情况下。因为没有读操作，第一次循环检查 `eof()` 返回失败。然后进入循环中我们读取文件，文件末尾的条件被触发。大那是现在太晚了，我们输出了数字 `1`，但是这个问题是空的。同样的逻辑，程序总会输出一行伪造额外的一行。
为了写出正确的程序，你需要在读操作操作后调用 `eof()` 函数。如果你不考虑到异常情况仅仅是文件末尾，你可以这些编写程序：
```cpp
int main() {
    ifstream ifs( "afile.txt" );
    int n = 0;
    string line;
    while( getline( ifs, line ) ) {
        cout << ++n << " " << line << '\n';
    }
}
```
在这里对 `getline()` 它将文件流当做第一个参数的函数的返回值进行了转换操作，用来测试 `While` 循环，这个循环将一直迭代直到文件末尾条件发生。
同样的 C 代码中，你不能这样编写代码：
```c
#include <stdio.h>
int main() {
    FILE * f = fopen( "afile.txt", "r" );
    char line[100];
    int n = 0;
    while( ! feof( f ) ) {
        fgets( line, 100, f );
        printf( "%d %s", ++n, line );
    }
    fclose( f );
}
```
对于空文件，这个代码同样也会输出一些打印出一些无效的内容，你应当这样做
```c
int main() {
    FILE * f = fopen( "afile.txt", "r" );
    char line[100];
    int n = 0;
    while( fgets( line, 100, f ) != NULL ) {
        printf( "%d %s", ++n, line );
    }
    fclose( f );
}
```
看上去 `eof()` 和 `feof()` 没有任何作用，那为什么 `C/C++` 标准库中提供他们呢？它们使用来判断文件读写是否异常发生而不是文件末尾条件发生，如果你想要区分它们，这样使用 `eof` 或者 `feof()`函数
```cpp
#include <iostream>
#include <fstream>
#include <string>
using namespace std;

int main() {
    ifstream ifs( "afile.txt" );
    int n = 0;
    string line;
    while( getline( ifs, line ) ) {
        cout << ++n << " " << line << '\n';
    }
    if ( ifs.eof() ) {
        // OK - EOF is an expected condition
    }
    else {
        // ERROR - we hit something other than EOF
    }
}
```
## 2.7 总结
看上去 `EOF` 非常复杂，但是遵循下面三个基本原则：
- 没有 `EOF` 字符，除非你在 `Windows` 下以文本的模式打开文件，或者自己实现文件读写；
- `C/C++` 中 `EOF` 不是文件结束符，它是在特定库函数中的特殊返回值。
- 不要循环 `eof()` 和 `feof()` 函数。

如果将这些规则记在心中，你应该能够避免由于误解 `C/C++` 文件结束标识符而导致的 Bug。
#  3 Tips

## 3.1 Shell 获取参数
- 全部参数： `$@` 或者 `$*`
- 参数长度： `$#`
- 第一个参数:  `$1`
- 最后一个参数：`${@:${#@}`
- 最后两个参数： `${@:@{#A}-1}`
- 倒数第二个参数： `${@:${#@}-1:1}`
- 从第二参数到最后一个参数： `${@:2}`
- 第二个参数开始，连续两个参数： `${@:2:2}`
# 4 Share
[如何变得成功？](http://blog.samaltman.com/how-to-be-successful)
这篇文章的标题非常鸡汤，但是观点却非常新颖，而且是可操作性的，而不是一味地强调人的主观能动性。
1. 复合式成长：大部分人的成长都是线性的，而指数级增长才是财富积累最快的方式。用 $72$ 除以增长的百分数，就可以得到在多少个周期内翻一倍。
2. 自信：这一点毋庸置疑，只有相信自己，才能看到真正的未来。大那是要注意平衡好自信和自我意思两者。
3. 独立思考：事业往往需要原创的思维方式，学校从来没有教授过独立思考的能力。
4. 擅长推销自己：自信是不够的，你需要让别人相信你。所有伟大的职业都是从推销开始的，销售需要很好的沟通能力。
5. 尝试去冒险：大部分人高估了风险而低估了收益。
6. 专注：专注于一件重要的事，避免时间浪费在无所谓的是事情上。
7. 努力工作：不管在任何领域，只要努力工作就可以超过 $90%$ 的人。
8. 胆大：做一些其他人不愿意尝试的事情。
9. 任性：有一个秘密就是世界会按照的你的的意愿发展，只要你足够愿意。
10. 变得有竞争力：打造个人品牌，使用他人来帮助你。
11. 人脉：与优秀的人交往，并且发挥每一个的长处而不是短处。
12. 资产才能变得富有：一个很大的误区是工资让你变得富有，而是资产才能让你快速积累财富，比如生意，房地产，知识产权等等。
13. 内驱：大部分人都是外在驱动的，跟随别人的脚本，只有跟随自己才能做到极致。