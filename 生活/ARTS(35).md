---
date: 2019-06-17
status: public
tags: ARTS
title: ARTS(35)
---

# 1 Algorithm

> 设计一个数据结构，在下面的操作时间复杂度都是 $O(1)$
> - `insert(val)`: 当元素 `val` 不存在，向集合中插入该项。
> - `remove(val)`: 当元素 `val` 存在，从集合中删除该项。
> - `getRandom`: 随机返回集合中一项，每个元素都有相同的概率返回。

对于 `insert` 和 `remove` 操作，使用 `map` 数据结构可以很容易实现；但是 `getRandom` 操作不能在 $O(1)$ 时间复杂度完成，因此要借助数组才能满足条件。
![](./_image/2019-06-18-16-56-54.jpg?r=53)

上图为依次往集合中添加 `6、10、1、5、9` 元素，下面是数组，用来依次保存数据；上面是一个字典，`key` 为待插入元素，`value` 为该值在数组中的索引。


- insert 操作
插入元素首先在字典中查找，如果存在直接返回；否则在数组后面直接添加一个元素，并且在字典中添加相应的键值。

- getRandom 操作
随机获取元素比较简单，直接从数组中随机选择一个元素返回。

- remove 操作

删除操作就比较单独处理一下，如果要删除元素 `6`，现在字典中查找出索引为 `0`，然后与数组中最后一个元素 `9` 交换，然后更新字典中 `9` 对应的值为 `0`
![](./_image/2019-06-18-17-30-06.jpg?r=50)
现在待删除的元素 `6` 已经移动到数组末尾，现在就可以在 $O(1)$ 时间复杂度内删除数组和字典的词条。

```go
type RandomizedSet struct {
	values []int
	index  map[int]int
	size   int
}

/** Initialize your data structure here. */
func Constructor() RandomizedSet {
	return RandomizedSet{
		values: make([]int, 0),
		index:  make(map[int]int, 0),
	}
}

/** Inserts a value to the set. Returns true if the set did not already contain the specified element. */
func (this *RandomizedSet) Insert(val int) bool {
	if _, ok := this.index[val]; ok {
		return false
	} else {
		this.values = append(this.values, val)
		this.index[val] = len(this.values) - 1
		return true
	}
}

/** Removes a value from the set. Returns true if the set contained the specified element. */
func (this *RandomizedSet) Remove(val int) bool {
	if idx, ok := this.index[val]; ok {
		lastValue := this.values[len(this.values)-1]
		this.index[lastValue] = idx
		this.values[idx] = lastValue
		delete(this.index, val)
		this.values = this.values[:len(this.values)-1]
		return true
	} else {
		return false
	}
}

/** Get a random element from the set. */
func (this *RandomizedSet) GetRandom() int {
	idx := rand.Intn(len(this.values))
	return this.values[idx]

}
```
# 2 Review
[To ORM or not to ORM](https://eli.thegreenplace.net/2019/to-orm-or-not-to-orm/)

**是否使用 ORM **
在于数据库打交道的过程中，使用 `database/sql` 包已经很久了。最近提到的 `gorm` 引起了我的对于使用 `ORM` 和 `database/sql` 比较的好奇心。由于之前有过使用 `ORM` 的经历，我决定开始尝试通过实际的例子来做个实验，比较是否使用 `ORM` 的区别和花费的精力。
这个实验让我写下了关于 `ORM` 的好处和缺陷的思考，如果你对这个感兴趣，可以阅读下面的内容。
我的实验是定义一个博客引擎的简单的数据库，同时写下一些 `GO` 代码来完成数据的查询和修改操作，分别用原生的 `SQL` 语句和 `ORM` 框架相比较。
下面是数据库的范式
![](./_image/2019-06-23-10-41-47.png)

虽然非常简单，但是这个范式包含了大部分构建简单的 `wiki` 和博客应用程序需要的全部元素，它既有 `1` 对多关系（博客和评论之间关系）和多对多的关系（博客和标签之间关系）。如果你更喜欢用 `SQL` 描述数据库范式，下面就是其中的定义。
```sql
create table Post (
    postID integer primary key,
    published date,
    title text,
    content text
);

create table Comment (
    commentID integer primary key,
    postID integer,
    author text,
    published date,
    content text,

    -- One-to-many relationship between Post and Comment; each Comment
    -- references a Post it's logically attached to.
    foreign key(postID) references Post(postID)
);

create table Tag (
    tagID integer primary key,
    name text unique
);

-- Linking table for the many-to-many relationship between Tag and Post
create table PostTag (
    postID integer,
    tagID integer,

    foreign key(postID) references Post(postID),
    foreign key(tagID) references Tag(tagID)
);
```
上述的 `SQL` 语句已经在 `SQLite` 中测试通过，其他的关系型数据库可能需要一些额外的调整。当使用 `gorm` 就没有必要写 `SQL` 语句，而是定义 `objects` (实际上是结构)，内部包含一些魔法般的字段标记。
```go
type Post struct {
  gorm.Model
  Published time.Time
  Title     string
  Content   string
  Comments  []Comment `gorm:"foreignkey:PostID"`
  Tags      []*Tag    `gorm:"many2many:post_tags;"`
}

type Tag struct {
  gorm.Model
  Name  string
  Posts []*Post `gorm:"many2many:post_tags;"`
}

type Comment struct {
  gorm.Model
  Author    string
  Published time.Time
  Content   string
  PostID    int64
}
```
[代码](https://github.com/eliben/code-for-blog/tree/master/2019/orm-vs-no-orm)操纵数据库有两种形式：
1. 非 `ORM`: 直接使用 `database/sql` 中的原生 `SQL` 查询语句。
2. `ORM`: 使用 `gorm` 库访问数据库。
所进行的操作如下：
1. 增加一些数据（博客、评论和标签）到数据库中；
2. 给定标签查询所有的博客；
3. 查询所有博客的细节（所有的评论和被标记的标签）

下面试任务 2 的两个变种实现方式：通过给定的标签查询所有的博客。首先是不使用 `ORM`:
```go
func dbAllPostsInTag(db *sql.DB, tagID int64) ([]post, error) {
  rows, err := db.Query(`
    select Post.postID, Post.published, Post.title, Post.content
    from Post
    inner join PostTag on Post.postID = PostTag.postID
    where PostTag.tagID = ?`, tagID)
  if err != nil {
    return nil, err
  }
  var posts []post
  for rows.Next() {
    var p post
    err = rows.Scan(&p.Id, &p.Published, &p.Title, &p.Content)
    if err != nil {
      return nil, err
    }
    posts = append(posts, p)
  }
  return posts, nil
}
```
如果你知道 `SQL`，这个实现非常直接。我们在 `Post` 和 `PostTag` 表之间做了一次内交集。然后通过 `tag ID` 进行过滤，剩下的代码就是迭代查询到的结果。
接下来就是 `ORM`:
```go
func allPostsInTag(db *gorm.DB, t *Tag) ([]Post, error) {
  var posts []Post
  r := db.Model(t).Related(&posts, "Posts")
  if r.Error != nil {
    return nil, r.Error
  }
  return posts, nil
}
```
在 `ORM` 代码中，我们倾向于直接使用对象（在这里是 `Tag`）而不是它们的 `ID`。`gorm` 生成的 `SQL` 查询语句和我直接手动写的 `SQL` 语句非常相像。
除了为我们生成了 `SQL`, `gorm` 还提供更好的方式生成返回结果，在 `database/sql` 中的代码，我们需要显式地迭代最终的结果，扫描每一行然后填入各自的结构体重的字段。`gorm` 中的 `Related` 方法（还有其他类似的方法）能够自动生成扫描然后完成全部结果集。
非常惊喜地是 `gorm` 帮我省去了大量的代码，对于这种简单的查询使用 `gorm` 并不难，需要查看 `API` 文档来了解使用方法。唯一感觉不好的地方是这个例子中，建立博客和标签的多对多关系非常讲究，结构中的字段标签非常丑陋和神秘。
上面简单的实验例子问题是很难去调试系统的临界值，在简单的例子中工作非常完美，但是在如果在到达临界情况下会发生什么就变得非常有意思了，它是如何处理复杂的查询和数据库结构的呢？所以我尝试去 `Stack Overflow` 上查看，上面有很多观点与 `gorm` 相关的问题，通常是关于这层的复杂度相关的，接下来我将阐述一下这个观点。
在任何情况下，复杂的功能被封装到另一层将会有导致增加复杂度的风险。因为这会导致遗漏的抽象，也就是说封装的那一层不能完美地完成被封装的一层所做的工作，这就导致了程序员需要同时对这两层做相关的开发工作。
不幸的是，`gorm` 被怀疑有这样的问题。在 `Stack Overflow` 上有无穷尽的关于用户关于 `gorm` 复杂度问题斗争的问题，还有它本身的限制等等。还有一些让人恼怒的问题就如你期望的（比如 `SQL` 查询的错误）。
从我们实验中可知使用 `ORM` 最大的优势是它帮我们省去了大量冗长的代码。大部分是关于是数据库相关的，这个在大部分应用程序中是相同的。
另一个不明显的优势是它抽象了不同数据库实现的后端。这个在 `Go` 中不是问题，因为`database/sql` 已经提供了很棒的可移植层。在一些缺乏标准 `SQL` 访问层中的语言中，这个优势就愈发明显了。
但是也有一些缺陷:
1. 需要在学习 `ORM` 层，包含了不同的语法、魔法般的标签等等。如果你对 `SQL` 本身非常熟悉，这一点是非常大的缺陷。
2. 即使你对 `SQL` 没有任何经验，也需要大量空白知识需要填补。任何单个 `ORM` 都是更复杂的知识，而且并不是非常通用。你将要花费大量的时间用来搞清楚它是如何运作的。
3. 调试查询效率是一个挑战，因为我们将抽象层次提高了。有时候需要我们一些特殊技巧以便让 `ORM` 框架生成正确的查询语句，但是有时候当你已经确切知道那种查询最快而使用 `ORM` 让人非常崩溃。
最后一点的缺陷要从长期角度来看：这些年来 `SQL` 变化非常小，特定语言的 `ORM` 往往一直是出现和消失。每一个流行的语言都有大量的 `ORM` 框架可供选择。当你在团队、公司和项目跳转，你或许要进行  `ORM` 框架之间切换，这是额外的工作负担。当你切换语言的时候，`SQL` 却是非常稳定的一层，它能帮你跨团队、语言和项目。
**结论**
在实现了简单的应用程序框架，分别使用原生 `SQL` 和对比 `gorm` 框架之后，我可以看到 `ORM` 的吸引力在下降。回想起当多年以前我是一个数据库新手，在使用 `Django` 中的 `ORM` 框架开发应用程序，这种感觉棒极了。我不需要考虑 `SQL` 或者下面的数据库，它就能工作了，但是使用的情形非常简单。
随着我的经历增加，我可以看到使用 `ORM` 的缺陷。尤其我认为在类似 `Go` 这样的语言，因为它有已经设计好 `SQL` 接口来屏蔽不同数据库后端实现。我宁愿花时间在编码而不是在阅读 `ORM` 的文档，优化我的查询，尤其是用来进行 debug。
我想 `ORM` 在 `Go` 中仍然会发挥作用，如果是大量的 `CRUD` 相关的应用程序，它能省下大量的时间。最后有需要提及文章 [Benefits of dependencies in software projects as a function of effort](https://eli.thegreenplace.net/2017/benefits-of-dependencies-in-software-projects-as-a-function-of-effort/) 在一些项目中关注重点是项目而不是数据库接口代码，也就是说工作内容不仅仅是 `CRUD`，那么 `ORM` 框架是不值得的依赖。
# 3 Tips
在设计 HTTP 服务的时候，接受请求的响应时间和响应体的大小有单独的设置，不能确定是否默认值。
# 4 Share