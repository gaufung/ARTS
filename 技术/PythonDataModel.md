---
date:  2017-09-07
title: Python数据类型
status: public
---
本文为Python3官方文档[Data Model](https://docs.python.org/3/reference/datamodel.html)的阅读整理。
# 1 Object
object是Python所有的抽象数据类型，每个对象一旦建立，其唯一标识符不会改变。在CPython实现中，使用`id(x)`返回x对象在内存中的地址。而且object对象不是显示被销毁，通常是由垃圾回收机制处理。在CPython中使用引用计数(reference-counting)机制来探测对象是否可达。
# 2 标准类型继承
## 2.1 None
内置数据类型 `None`
## 2.2 NotImplemented
内置数据类型 `NotImplemented`
## 2.3 Ellipsis
字面类型为`...`
##  2.4 numbers.Number
包含数学运算的数据类型，主要包含

+ number.s.Integral 整数类型
     - int (unlimited range)
     - boolean (True or False -> 1 or 0)
     - float(double precision)
     - complex(complex number)

## 2.5 Sequences
内置函数`len()`能够返回序列元素的数量
## 2.5.1 Immutable Sequence
+ Strings

字符串数据类型代表了Unicode字符，所有字符的Uncidoe值在`U+0000~U+10FFFF`之间。Python不支持`char`数据类型，内置函数`ord()`将一个字符串转换为到`0~10FFFF`,而`chr()`函数将一个`0~10FFFF`转换到相应的长度为1的字符串类型。而`str.encode()`将一个字符串转为一个字节类型，而`byte.decode()`功能与之相反。

+ Tuples

使用圆括号包含起来的对象

+ Bytes

使用`byte`数据类型构成的不可变数组，其中每个对象为`8-bit`的字节，范围为`0<=x<256`。

### 2.5.2 Mutable Sequence
+ Lists

数据类型可以为Python任意对象

+ Byte Arrays

可变的字节数组

## 2.6 Set 
+ Sets

可变的数据集合

+ Forzon sets

不可变的数据集合

## 2.7 Mappings
Dictionay字典类型可以存储任何Python数据类型，不过Key类型是不可变的。

## 2.8 Callable types
可调用类型主要有

### 2.8.1 用户定义函数
用户数据类型包含了一下特殊的属性

AAttribute  | Meaning
 ---  |  ---
__doc__ | fucntion documents
__name__ | function name
__module__ | name of the module
__defaults__ | a tuple containing the default arguments
__code__ | compiled function body
__kwdefaults__ | a dict containing defaults for keyword-only parameters


而且用户定义函数也支持自定义任意属性。
### 2.8.2 实例方法

+ __self__: 实例对象
+ __func__: 方法对象
+ __doc__: 方法文档
+ __module__:模块名称

当实例方法被调用的时候，相当于`__func__`方法被调用，并且将`__self__`插入到参数前面。
```python
class MyClass(object):
    def foo(self, a):
        print(a)
mc = MyClass()
```
那么`mc.foo(1)`相当于`MyClass.foo(self, 1)`

### 2.8.3 生成函数
包含 `yield`的方法

### 2.8.4 协程函数
使用`async def`和`await`修饰的函数
### 2.8.5 内置函数
诸如`len()`等C语言封装好的函数
### 2.8.6 类
类也是可调用的，当创建一个实例的时候，调用`__new__()`方法创建一个实例，使用`__init__()`方法初始化这个实例。
### 2.8.7 类对象实例
如果该类定义了`__call__()`方法，那么这个类对象是可调用的。

## 2.9 Modules
模块是Python代码基本组织结构，模块可以自定义一些属性(attributes)，对于模块`m`，那么`m.x=1`相当于`m.__dict__['x']=1`
## 2.10 Custom Class
用户自定义的类可以使用`class`等关键字创建，每个`class`的命名空间由一个字典对象构成，如果`C`为一个类，那么`C.x=1`相当于`C.__dict__['x']=1`，再查找的过程中，如果一个类没有该key，那么将会去其基类型查找，搜索的方式采用`C3 Method`。
## 2.10 Class Instances
对象实例的命名空间也是由一个字典组成，如果属性没有对象字典中没有找到，那么将会去类的命名空间中查找。

## 2.11 I/O Object
`sys.stdin`，`sys.stdout`和`sys.stderr`分别对应于解释器的标准输入、输出和错误输出。

# 3 特殊方法名
## 3.1 __new__, __init__, __del__
在Python中创建一个对象，首先是调用`cls.__new__`方法创建一个实例，然后调用`self.__init__`方法对其进行初始化工作。在创建对象实例的时候，通常也需要调用基类的`__new__`方法，同样初始化对象的时候，也需要调用基类的`__init__`方法。因此Python的单例设计模式
```python
class Singleton(object):
    def __new__(cls, *args, **kw):
        if not hasattr(cls, '_instance'):
            cls._instance = super(cls, Singleton).__new__(cls, *args, **kw)
        return cls._instance
# custom class
class MyClass(Singletion):
    pass
```
而`__del__`定义了函数在销毁的时候调用，但是由于CPython采用的垃圾回收机制，因此只有某个对象的引用计数减少到0的时候才会调用。
## 3.2 __format__
自定义`format`格式化输出
## 3.3 __eq__, __hash__
用户自定义的类默认实现了`__eq__()`和`__hash__()`，如果重载了`__eq__()`方法，那么必须重载`__hash__()`方法
## 3.4 __bool__
当布尔值判断时候或者内置函数`bool()`时候调用, 如果没有定义，那么`__len__`方法将会被调用，如果两者都没有定义，那么将会返回`True`。

# 4 自定义属性访问
## 4.1 __getattr__, __setattr__, __delattr__
控制实例对象的属性访问控制， 当一个对象的属性没有找到的时候，将会调用`__getattr__`方法，同样对于给定一个属性，并赋予一个值调用`__setattr__`方法。通过扩展字典功能，使得字典能够支持key属性。
```python
class Dict(dict):
    def __getattr__(self, name):
        try:
            return self[name]
        except KeyError as err:
            return None
   def __setattr__(self, name, value):
       self[name]=value

d = Dict()
d.name = 'gaufung'
print(d.name)
>> gaufung
```

## 4.2 __dir__
当内置函数`dir()`调用该对象的时候，该方法被调用，并且返回一个集合。

## 4.3 __solts__
规定了对象实例包含的属性个数和名称，也就是字典key的槽位。