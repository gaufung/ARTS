---
date: 2016-12-17 14:38
status: public
title: 'Python 奇淫技巧'
---

# 1 字符串拼接
Python 语言中字符串的操作与 C# 和 Java类似，每次操作将会生成一个新的字符串，如果出现较多的字符串拼接，如果直接使用 `+=` 操作，将会影响程序的性能，建议使用 `join()`操作
```Python
def combine_str(n):
    a = []
    for i in range(n):
        a.append('a')
    return ''.join(a)
```

# 2 列表推导
列表推导是 Python 列表的一大利器，使用列表推导可以避免难看的循环操作
## 列表操作
```Python
a = range(10)
b = [item for item in a if item % 2 ==0]
```

## 字典操作
```Python
a = ['name','university','year']
b = ['gaufung','cumt','1992']
result = {k:v for k,v in zip(a,b)}
```

# 3 迭代循环
对于列表的循环操作，可以不适用`for i in range(len(list))`这种方式来迭代内容，可以使用 `enumerate` 方式，如果对列表内
```Python
a = range(10)
for idx,item in enumerate(a):
    pass
for _,item in enuerate(a):
    pass
```

# 4 输出控制
当对某个列表进行输出控制，为了能够在一行中输出结果，可以在`print `语句后加入`,`
```Python
a = range(5)
for item in a:
    print item,
>>> 0 1 2 3 4
```