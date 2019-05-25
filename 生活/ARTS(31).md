---
date: 2019-05-19
status: public
tags: ARTS
title: ARTS(31)
---

# 1 Algorithm

> 给定一个 `IPv4` 地址的字符串，将其转换为 `int` 表示的整数，需要考虑不合法的字符串，只能使用常数空间。比如字符串 `127.0.0.1` 表示一个 `IPv4`地址，每一段都是 `byte` 表示的整数，四个 `byte` 可以表示 32 位的整数 `2130706433`。

首先对字符串按字符 `.` 划分为四个部分，在将其转换为整数，然后依次左移 `24`, `16`, `8` 和 `0` 位，将这些左移后的结果按照**或**(`|`)操作完整整数转换。那么不合法的情况有哪些呢？

- 合法的 `IPv4` 地址每一段的范围只能是 $[0, 255]$；
- 按照 `.` 划分只能划分为四个部分；
- 字符串只能包含数字和 `.` 两种字符；
- `.` 不能连续两次出现，也不能出现在字符串首和尾。

```go
func IPv4ToInt(s string) (int, error) {
    i := 0
    dotCount := 0
    res := 0
    val : = 0
    for i < len(s) {
        if s[i] == '.'{
            if i==0 || (i>0 && s[i] == s[i-1]) || i == len(s)-1 {
                return 0, errors.New("invalid IPv4")
            }
            if dotCount >= 3 {
                 return 0, errors.New("invalid IPv4")
            }
            val = val << uint(8 * (3 - dotCount))
            res = res | val
            val = 0
            dotCount ++ 
        }else{
            if !(s[i] >= '0' &&s[i] <= '9') {
                return 0, errors.New("invalid IPv4")
            }
            val = val * 10 + int(s[i] - '0')
            if val > 255 {
                return 0, errors.New("invalid IPv4")
            }
        }
        i++
    }
     if val > 255 || dotCount > 3 {
        return 0, errors.New("invalid IPv4")
    }
    res = res | val
    return res, nil
}
```

# 2 Review

[Technical Debt](https://martinfowler.com/bliki/TechnicalDebt.html)

**技术债**

软件系统往往会构建在缺陷之上的，缺乏内部质量导致为系统修改和增加新的功能变得更加困难。有一种很好的比喻就是*技术债*，它是由 `Ward Cunningham` 提出来的，用来形容我们如何面对这种缺陷。就跟债务一样，我们为系统增加新功能而做的额外的工作就是为这些技术债而付的利息。

![debt](./_image/2019-05-22-11-31-44.jpg?r=60)

想象一下代码库中有一个非常令人困惑的模块，这时我需要增加新的功能。如果这个模块结构非常清晰，那么需要花费 4 天时间来完成工作。但是由于模块中缺陷，需要花费我 6 天时间，这两天的的差别就是技术债的利息。
这个技术债比喻最吸引我的地方是它告诉我如何处理这些缺陷。我可能花费 5 天时间清理模块结构，移除缺陷，就像还清本金一样。如果我仅仅是为了一个功能，这样做没有任何收益，因为将要花费 9 天而不是 6 天。但是如果接下来有更多相似的功能需求，在移除缺陷之后，这些新工作是非常高效地完成。
说是这样说，听上去好像仅仅是数字上运算的结果，任何经理都可以通过 Excel 来计算并做出选择。不幸的是我们[不能计算生产效率](https://martinfowler.com/bliki/CannotMeasureProductivity.html)，上述的任何成本都不能客观地测量。虽然我们可以估计它需要完成功能需要多长时间；估计移除上述缺陷的影响；估计需要花多长时间来移缺陷，但是我们估计的准确性非常低。
考虑到我们针对债务的方式一样，一步步偿清本金。在第一个功能的时候，我将会花费几天时间来移除一些缺陷，它可以减少在将来某一天的带来的*利息*。虽然还需要花费额外的时间，但是通过移除那些将来改变的代码中缺陷。这种渐进式改进的好处是我们可以花更多的时间在移除那些我们经常修改的模块中，它们也是我们代码库中我们需要被移除的不好设计的地方。
将付利息和付本金考虑对比考虑可以帮助我们决定那一部分的设计优先解决。如果我的代码库中有一个糟糕的部分，每一次修改都是一场噩梦，但是果我没有修改就没有问题。我仅仅是在软件开发过程的一部分工作，才需要引发支付*利息*（在这里比喻并不是非常精确，因为支付财务利息是由时间长短决定的）。虽然这部分代码如此糟糕，但是只要不管它也能相安无事。与此相反的是，经常性修改的代码需要零容忍的态度，因为为它们支付利息是非常高的。它们是非常重要的，因为如果开发者不关注内部质量，这些缺陷就会累积起来，越多的改变，缺陷累积的风险越大。
技术债的比喻也常常用来为忽视内部质量行为作辩解，论据是移除这些缺陷需要花费大量的时间和精力。如果有紧急的需求，或许最好的办法就是接受这个技术债，而且默默告诉自己在将来的某一天会重新整理它们。
危险的地方是，大部分时候上述这些分析都是错误的。缺陷有很快速的影响力，会拖慢每一个需要紧急完成功能。如果一个团队这样做的话，将会为支付技术债而刷爆他们的信用卡，最后还需要花费大量精力来提高软件内部的质量。这种比喻往往也会导致开发者不知所措，因为不断改变的现实状况和金融贷款情形并不完全一致。承担这些*债务*来加速软件交付只有在你们还处理偿还线以下时候才起作用，而且这样做的话你们会在几周而不是几个月之后就会碰到这根线。
还有一种争论是是不是不同种类的缺陷都可以被认为是技术债。我认为最好的方法是思考一下技术债是否是刻意要求还有它是否认真思考或者随意思考的。进一步阅读[参考文章](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html)。

**进一步阅读**

- 我所了解的，Ward 第一次提出这个概念是在 [1992 年 OOPSLA](http://c2.com/doc/oopsla92.html)经验分享会上。同样也被作为 [Wiki](http://wiki.c2.com/?ComplexityAsDebt)被讨论。
- Ward Cunningham 曾经有一个[视频演讲](https://www.youtube.com/watch?v=pqeJFYwnkjE)来讨论过他创建的比喻。
- Dave Nicolette 通过[案例](http://neopragma.com/index.php/2019/03/30/technical-debt-the-man-the-metaphor-the-message/)来拓展 Ward 关于技术债的观点。
- 一些读者想出了一些类似的好名称。David Panariti 将丑陋的编程称之为 *赤字编程*，显然他是根据政府的政策来命名的，我想现在更加自然而然了。
- Scott Wood 建议使用 *技术通胀* 来形容这种情况，因为当前技术水平已经超出之前设计的时候，而且它和当前的工业水平失去了兼容性。比如你的语言版本已经很落后主流版本，你的代码将不能适应于主流的编译器。
- Steve McConnell 为这个比喻提出好几个观点，尤其是如何控制你无意识的技术债而为那些不得不做的技术债保留回旋余地。我同样喜欢他的最小支付的概念。
- Aaron Erickson 讨论过关于[注册金融](http://www.informit.com/articles/article.aspx?p=1401640)
- [Henrik Kniberg](http://blog.crisp.se/2013/10/11/henrikkniberg/good-and-bad-technical-debt)讨论过更老的技术债会导致更大的问题，所以很有必要去管理这些。
- Erik Dietrich 讨论过[技术债的人力成本](http://www.daedtech.com/human-cost-tech-debt/): 团队内讧、技术的延迟和内耗。

# 3 Tips

Linux 文本处理

## 3.1 find 

查找 txt 和 pdf 文件：`find . \(-name "*.txt" -name "*.pdf" \) -print`

正则查找： `find . -regex ".*\(\.txt|\.pdf\)$`

否定参数，查找非 txt 文件： `find . ! -name "*.txt" -print`

## 3.2 grep

`grep match_pattern file` 按行查找匹配行

- o 输出匹配的文本行
- v 输出不匹配的文本行
- c 统计出现的次数
- i 忽略大小写

## 3.3 xargs

将输入的数据转换为特定命令的命令行参数。

- d 定义定界符
- n 输出特定的行数
- I {} 指定替换字符串
- 0：指定0为输入定界符

## 3.4 sort

`sort -nrk 1 data.txt` 对 `data.txt` 文件排序

- n 数字排序
- d 字典排序
- r 逆序排序
- k N 按照第 N 列排序

## 3.5 uniq

`sort unsort.txt | uniq` 消除重复行

- c 统计出现的次数
- d 找出重复行

## 3.6 tr

`echo 12345 | tr '0-9' '987654321'` 将 `12345` 转换为 `87654`

**可用的字符**

- alnum：字母和数字
- alpha：字母
- digit：数字
- space：空白字符
- lower：小写
- upper：大写
- cntrl：控制（非可打印）字符
- print：可打印字符

`echo Linux | tr '[:lower:]' '[:upper:]'` 输出 `LINUX`

## 3.7 cut 

截取文件

- `cut -f2, 4 filename.txt` 截取 2 到 4 列
- `cut -f2 -d ";" filename.txt` 截取 2 列， 以 `;` 为定界符
- b 以字节为单位
- c 以字符为单位
- f 以字段为单位

## 3.8 paste

将两个文本按照列拼接在一起

`paste file1 file2`

- d 指定定界符

## 3.9 wc

- wc -l file 统计行
- wc -w file 统计词
- wc -c file 统计字符

# 4 Share

[China, Leverage, and Values](https://stratechery.com/2019/china-leverage-and-values/)

这篇文章对彭博社[技术铁幕已经拉起](https://www.bloomberg.com/opinion/articles/2019-05-20/huawei-supply-freeze-points-to-u-s-china-tech-cold-war)这篇文章的评论，主要有一下几个观点

- 美国在高科技领域领先中国大陆是毫无疑问的，而且也没有短时间追的上的可能；
- 中国大陆在 WTO 之后，广泛引进国外的科技，比如芯片、操作系统和基础设施，但是并没有开放国外软件和很多互联网服务，比如 `google`, `facebook`。这些保护主义措施促进了国内的互联网发展，并培养出很多巨头，而且这些巨头也开始向海外进军。
- 互联网服务本质上是信息流，而中国大陆政府对信息的掌控更加敏感。