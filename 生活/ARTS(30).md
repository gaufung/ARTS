---
date: 2019-05-12
status: public
tags: ARTS
title: ARTS(30)
---

# 1 Algorithm
>![](./_image/2019-05-12-10-31-05.jpg?r=60)
>上图为一手机键盘，手指尖可以在键与键之间移动，但是每次只能按照`日`字对角线移动，类似中国象棋`马`移动方式。现在输入指尖开始的键( `key` )和可以移动的步数 `k`，请输出移动最终移动到 `1` 键方法数。

按照移动规则，将键盘转换为网络连接图，按照`日`字关系绘制跳转关系。为了方便，我们将全部键按照一维数组形式表示出来。
```go
// 连接表形式表示图
func buildGraph(m map[byte]int) [][]int {
    neighbour := make([][]int, len(m))
    neighbour[m['1']] = []int{ m['6'], m['8']}
    neighbour[m['2']] = []int{ m['7'], m['9']}
    neighbour[m['3']] = []int{ m['4'], m['8']}
    neighbour[m['4']] = []int{ m['3'], m['0'], m['9']}
    neighbour[m['5']] = []int{ m['*'], m['#']}
    neighbour[m['6']] = []int{ m['1'], m['7'],  m['0']}
    neighbour[m['7']] = []int{ m['2'], m['6'], m['#']}
    neighbour[m['8']] = []int{ m['1'], m['3']}
    neighbour[m['9']] = []int{ m['2'], m['4'], m['*']}
    neighbour[m['*']] = []int{ m['5'], m['9']}
    neighbour[m['0']] = []int{ m['4'], m['6']}
    neighbour[m['#']] = []int{ m['5'], m['7']}
    return m
}
func buildMap() ma[byte]int {
    return map[byte]int {
        '1':0,
        '2':1,
        '3':2,
        '4':3,
        '5':4,
        '6':5,
        '7':6,
        '8':7,
        '9':8,
        '*':9,
        '0':10,
        '#':11,
    }
}
```
## 1.1 BFS 搜索
类似 BFS 搜索方式，每增加一次移动，保存 12 个键能够到达的数量。借助一个队列保存该上一步能够到达键的位置。
```go
func comeToOne(start byte, k int) int {
    mapper := buildMap()
    graph := buildGraph(mapper)
    index := mapper[start]
    queue := map[int]bool {
        index:true,
    }
    counter := make([]int, 12)
    counter[index] =  1
    for k > 1 {
        k--
        newCounter := make([]int, 12)
        newQueue := make(map[int]bool, 0)
        for k, _ := range queue {
            for _, neighbour := range graph[k] {
                newCounter[neighbour] += counter[k]
                if _, ok := newQueue[neighbour]; !ok {
                    newQueue[neighbour] = true
                }
            }
        }
        counter = newCounter
        queue = newQueue
    }
    return counter[mapper['1']]
}
```
`for` 循环表示移动的次数，每次根据上一步的能够到达的键更新下一步能够移动到的键，并累加移动的方法数。

## 1.2 递归
既然要求得在第 `k` 步到达 `0` 的位置的种类，那么也就是在 `k-1` 步到达 `6` 和 `8` 的种类之和，那么可以递归的执行下去，直到 `k==0` 的时候，除了 `0` 键，其余的键值都为 `0`
```go
func comeToOne(start byte, k int) int {
    return jump('1', start, k)
}
func jump(to, start byte, k int) int {
    if k == 0 {
        if to == start {
            return 1
        }else{
            return 0
        }
    }
    if to == '0' {
        return jump('4', start,  k-1) + jump('6', start, k-1)
    }else if to == '1' {
        return jump('6', start,  k-1) + jump('8',  start, k-1)
    }else if to == '2' {
        return jump('7', start,  k-1) + jump('9', start,  k -1)
    }else if to == '3' {
        return jump('4', start,  k-1) + jump('8', start,  k-1)
    }else if to == '4' {
        return jump('3', start,  k-1) + jump('9', start,  k-1) + jump('0',  start, k-1)
    }else if to == '5' {
        return jump('*', start,  k-1) + jump('#', start,  k-1)
    }else if to == '6' {
        return jump('1', start,  k-1) + jump('7', start,  k-1) + jump('0', start,  k-1)
    }else if to=='7' {
        return jump('2', start,  k-1) + jump('6',  start, k-1) + jump('#', start,  k-1)
    }else if to == '8' {
        return jump('1', start,  k-1) + jump('3', start,  k-1)
    }else if to == '9' {
        return jump('1', start,  k-1) + jump('2', start,  k-1) + jump('*', start,  k-1)
    }else if to =='*' {
        return jump('5', start,  k-1) + jump('9',  start, k-1)
    }else {
        return jump('5', start,  k-1) + jump('7', start,  k-1)
    }
}
```
## 1.3 动态规划
递归版本和递归求解 `Fibnacci` 数值比类似，时间复杂度将达到 $O(2^n)$，这个结果是不可接受的，因此结合 BFS 版本，实现动态规划版本。
```go
func comeToOne(start byte, k int) int {
    mapper := buildMap()
    graph := buildGraph(mapper)
    dp := make([][]int, k)
    for i:=0; i < k; i++{
        dp[i] = make([]int, 12)
    }
    startIndex  := mapper[start]
    dp[0][startIndex] = 1
    for i:=1; i < k; i++{
        for j:=0; j < 12; j++ {
            neighbours := graph[j]
            cnt := 0
            for _, neighbour:= range neighbours {
                cnt += dp[i-1][neighbour]
            }
            dp[i][j] = cnt
        }
    }
    return dp[k-1][mapper['1']]
}
```

# 2 Review
[The Unreasonable Effectiveness of SQL](https://blog.couchbase.com/unreasonable-effectiveness-of-sql/)
**SQL 语言不可思议的高效性**
25 年前两个年轻的 IBM 研究员带来第四代关系数据库操作数据新的语言，它是声明式而且非常容易使用。在 `Don Chamberlin` 和 `Ramond Boyce` 发表了 [SEQUEL: A Strucutred English Query Language](https://researcher.watson.ibm.com/researcher/files/us-dchamber/sequel-1974.pdf) 之后几年，`SQL` 获得了长足的发展并且适应了大量的技术: `OLTP`, `OLAP`, 对象数据库，对象关系数据库甚至 `NoSQL`。`SQL` 还启发了很多非关系型数据库查询语言的设计，比如对象数据库 `SQL`, 对象关系 `SQL`, `XML`语言 `SQL`，空间查询 `SQL`, 搜索 `SQL`, `JSON` 数据 `SQL`, 时间序列的 `SQL`，流数据`SQL` 等等。甚至像 `BI` 工具也会和不同的 `SQL` 进行交互。事实上，`SQL` 是第四代语言语言中最成功的。

> SQL 最神奇之处已经它超出了它的能力 -- `Lukas Eder`

正如最近 `Don` 所说的， `SQL` 基础是关系代数，它的目标是通过提供类似英语查询达到易用性的目的：
- 声明式语言和处理（而不是命令过程式）
- 通过组装语句写出复杂的查询
- 和 `Edger F Codd` 提出的关系模型相适应

尽管大数据仓库已经不再使用关系数据库系统，但是它们仍然尽力使用同样的 `SQL`，比如 `Hive`, `Impala`,  `drill`, `BigSQL`，它们都是受到 `SQL` 的启发，优化器和执行器都和 `SQL` 执行相类似。在每一个你能想到的数据存储和模型中，它们还在有规律地为 `SQL` 增加新的功能。将数据存储的格式、模型和 `SQL` 查询过程分隔开来带来了很大的好处。在 `SQL` 引入二十几年来，很多种数据库都消失了，很多数据处理流程也消失了。尽管在一些 `NoSQL` 浪潮中或多或少在暗示了 `SQL` 和 `SQL` 数据库已经消亡了，但是 `SQL` 阵营仍然大步流星前进。最近 `Don Chamberlin` 说：“如果一个语言是如此引人注目以至于其他语言证明自己和它不同，那么这个语言是非常了不起的"
换句话说现代数据库往 `No-SQL` 方向发展。虽然现在的定义是 `No Only SQL`, 但是最初的设计是要抛弃 `SQL` 并且尝试使用替代的语言和框架比如 `mapreduce` 来处理数据。但是十年过去了，每一个流行的 `NoSQL` 语句库都有一个 `SQL` 的变种， 比如 `Couchbase` 的 `N1QL`, `Cassandra` 的 `CQL`, `ElasticSearch` 中的 `Elastic`。你可能会说 `MongoDB` 没有 `SQL`, 但是我的回答是 `Squint!`, 你会看到这是一种简单的 `SQL` 的实现，通过在 `MongoDB` 中使用简单的过程式设计，查询一些松散的组合数据，而这些在 `SQL` 中已经早已经被实现。
尽管关系模型非常成功，但是数据库还是支持各种数据模型：`JSON`, `Graph`, `XML`, 时间序列、空间数据、宽列存储、列存储和文档存储等形式。但是这些每一种数据库都包含了各自的 `SQL` 语言版本，比如 `N1QL` 是 `JSON` 的 `SQL`, `SQL/XML`, `InfluxDB` 的 `SQL`, `SQL/Spatial`, `Cassandra` 数据库中的 `CQL`。甚至一些 `NoSQL` 数据库中也实现了 `SQL`。在最新流行的**数据科学**的世界中， `SQL`技能是被高度推荐掌握的技能。`Lukas Eder` 曾经在一次演讲中指出这一点，演讲的链接在附录中给出。

> 现在一些 `NoSQL` 数据库中比关系型数据库中包含更多的 `SQL`。

数据模型/格式 | SQL 实现
---|---
JSON	 | Couchbase N1QL: SQL for JSON
Wide column  | Cassandra CQL
Wide column  | Cassandra CQL
Hadoop/Big Data  | Hive, Impala, Drill, BigSQL
TimeSeries | 	Influxdb
Graph  | SQL Graph Database, Oracle Graph
NoSQL database  | Apache Phoenix
Spatial  | Oracle Spatial
Search  | Elastic SQL

**为什么 SQL 如此成功？**
- 声明式：你只需要声明输出内容，查询引擎会执行最优化的方式来查询。优化器尤其是 `Pat Selinger` 在 1979 年提出的基于成本分析的优化器，给性能带来了持续的提升。这也是现在出现的 `SQL` 重要的参考依据，最近发布的 `Apache Hive` 论文却是包含复杂和传统部分；
- `SQL` 不仅仅是适合查询，而且也是更新数据、执行事务、存储过程、用户自定义函数等等拓展，这是是过程式语言和声明式 `SQL` 的结合；
- `SQL` 是延续发展的，它已经被标准化很多次，每一次差不多增加一本书的内容、语句和大量的关键词。当然并不是所有的 `SQL` 都是一样的，甚至是传统基于关系数据库的传统的 `SQL` 都不是严格兼容的，除非你仔细写下这些兼容的 `SQL` 语句。尽管这些问题，`SQL` 的精髓还是保留下来。比如 `SQL` 下一个进化版本是 `SQL++`，`Don Chamberlin` 和 `Prof. Mike Carey` 讨论支持更富在的数据模型的需求，帮助用户和开发者更方便的访问 `JSON` 数据。`Don` 的书 《SQL++ 从入门到精通》介绍了最近 `SQL++` 的发展，为灵活的 `JSON` 数据提供方便的操作功能，但是这和 `SQL` 是兼容的。
- `SQL` 跟英语从其他语言借鉴一样，为新的数据类型、访问方法和使用方式都提供了拓展接口。
- `SQL` 同数据类型的独立允许它能够被用在非关系型数据：`CSV`, `JSON` 和所有的大数据格式。一些人将关系模型和 `SQL` 混淆为一体。事实上，在给定的格式下，`SQL` 允许你对任何数据格式进行 `select-join-group-aggregate-project` 操作。

**评估 SQL 支持**
既然 `SQL` 无处不在，你需要做的是它支持的级别。
1. 弄清楚工作的特点和工作的目标。比如交互式应用程序、自发性分析或者批处理分析或者 BI 工作内容等等；
2. 支持的语句反映了操作能力；
3. 语言在表示是的能力（`scalar`, `aggregate`,`boolean`), 连接 (`inner`, `left/right/full outer` )，子查询、表继承、排序和分页（`LIMIT/OFFSET`）；
4. 索引：没有正确的索引的 `SQL` 仅仅是图灵完备原型的语言；
5. 优化器：查询重写、选择正确的访问路径、创建优化查询路径这些功能让 `SQL` 成为成功的第四代语言。有些是基于规则的优化器，有些是基于成本的优化器，有些是基于两者的结合。评估优化器的的质量是非常严格的，传统的 `benchmarks` 不会起作用；
6. 正如一句谚语所说：在数据库中有三件非常重要的事情：性能、性能还是性能。在你的工作中测量性能是非常重要的，`YCSB` 和它的拓展 `YCSB-JSON` 可以让你的工作轻松点；
7. SDK：丰富的 `SDK` 和语言支持能够帮你快速完成开发工作；
8. BI 工具支持：对于大数据分析，通过标准数据库连接来支持 BI 工具是非常重要的的。

`N1QL` 发明者 `Gerald Sangudi` 曾经赞赏过 `SQL` 非常成功，因为它代表了数据处理的基本操作。`SQL` 支持丰富的操作集合 `select-join-nest-unnest-group-aggregate-having-window-order-paginate-set-ops`，这是我们或者机器在面对特定的数据思考的方式？但是下面的是可预见的，其他比如 `Python` 和 `Java` 语言为数据增加这些操作的功能。获取其他语言也能持续跟进。`SQL` 可能就不在了尽管关系模型任然存在，正如一句话所说的：
> SQL 已死，SQL 长存

**参考**
- [SQL++ For SQL Users: A Tutorial](https://www.amazon.com/SQL-Users-Tutorial-Don-Chamberlin/dp/0692184503/)
- [Lukas Eder – 2000 Lines of Java? Or 50 Lines of SQL? The Choice is Yours!](https://www.youtube.com/watch?v=S3JbAEd5u_c)
- [Ten SQL Tricks that You Didn’t Think Were Possible](https://www.youtube.com/watch?v=mgipNdAgQ3o)
- [How Modern SQL Databases Come up with Algorithms that You Would Have Never Dreamed Of by Lukas Eder](https://www.youtube.com/watch?v=wTPGW1PNy_Y)
- [SQL History](https://youtu.be/LAlDe1w7wxc)
- [Eugene Wigner: The unreasonable effectiveness of mathematics in the natural sciences](https://www.maths.ed.ac.uk/~v1ranick/papers/wigner.pdf)
- [Alon Halevy, Peter Norvig, and Fernando Pereira: The Unreasonable Effectiveness of Data](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35179.pdf)
- [Difference between 3GL and 4GL](https://www.reference.com/technology/difference-between-3gls-4gls-8826a78c1e34c2e)
# 3 Tips
**Linux 文件及目录管理**
- 以列表的方式显示目录项 `ls -lrt`
- 为文件增加一个 id 编号 `ls | cat -n`
- 查询文件或者目录 `find ./ -name 'core*' | xargs file`
- 递归当前目录并且删除特定的文件 `find ./ -name "*.o" -exec rm {} \`
- 查看文件并显示行号 `cat -n <file_name>`
- 显示文件的不同 `diff file1 file2`
- 查询文件的内容 `egrep '03.1\/CO\/AE' test.log`
- 建立硬连接 `ln cc ccAgain`
- 建立软连接 `ln -s cc ccTo`
- 管道连接 `|`; 串联执行 `;`; 前面执行成功才执行第二个 `&&`;  前面执行失败才执行第二个 `||`
- 清空文件 `:> a.txt`
- 删除光标到行首或者全部行： `ctl - u`
- 删除光标前的字 (`word`) : `ctl - w`
- 删除光标钱的字符(`character`): `ctl-h`
# 4 Share
前一段时间，`Oracle` [中国研发中心裁员新闻](https://36kr.com/p/5203117)被广泛发酵。外企、996、IT 等其他相关话题按下不表。为什么到科技如此发展的今天，求职者工作的年龄越发受到限制？比如我爷爷奶奶他们一辈，从十几岁在土地里劳作，一直到七八十岁还在耕作；而父母他们那一辈，如果进了工厂或者单位，工作到五六十岁差不多进入退休生活；但是到了我们这么一代，趋势却是到了四十岁，如果仍然领着一份工资生活，风险是非常高的，因为你随时会被企业或者单位裁掉。
在农业为主社会中，人口是非常重要的资产。因为和自然对比中，人是非常脆弱的，多一个人就能增加一份力量，比如多开垦一块地，更快地收购庄稼等等。
工业社会的来临，人不需要将全部重心同自然对抗中，工厂需要人操作机器完成生产，小时候家里附近的工厂按件计费将一部分细小的活分出来给周边农闲的人。这个时候机械的自动化程度还不高，都需要人的辅助工作，而且一天也只有 24 小时，有多大的厂房只能负载对应的机器和人力。扩展规模和经营也只能扩展机器和人力，边际效益不会发生变化，所以工厂会依赖更多工人。
现在技术发展飞快，尤其是互联网的发展，将可扩展性 (`scale`) 概念发挥的淋漓尽致，互联网服务一万个人和服务十万个人的成本几乎是一样的，这样给了互联网公司极大的想象力，同样地互联网公司也不需要雇佣更多的人来完成更大的规模效益。工厂自动化程度越来越高，较少的管理人员就能完成之前规模扩大的需求。
那么每一个工作的人需要这么怎么做才能避免被技术发展的社会抛弃呢？
- 拥有可扩展性（`scale`）的技能：除了附属在工作平台上的资源，属于自己能够可扩展性能力。比如写作、版权或者从事艺术相关无法被技术替代能力。
- 拥有资产：现如今投资的渠道是非常多，快速赚到人生的第一桶金并且购买能够给自己带来被动收入的资产。增产的增值才是获取财富的不二法门，而工资只需要保证正常的现金流。
- 获取资源：人除了工作能力，还包含其背后拥有的资源。尤其是社会资源，这是无法被机器或者科技所取代的。拥有这些资源就相当于拥有谈判的筹码。