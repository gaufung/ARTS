---
date: 2016-11-27 09:57
status: public
title: 'Python with 模块'
---

# 0 with 语法
```Python:n
with expression as target:
    pass
```
常见的文件操作
+  使用 try 操作
```Python:n
fp = open('my.dat', 'w')
try:
    fp.write('hello world!')
finally:
    fp.close()
```

+ 使用 with 操作
```Python:n
with open('my.dat','w') as fp:
    data = fp.write('hello world')
```
从直观上来看，使用 with 能够从客观上减少代码量
# 1 自定义 ContextManager 使用 with 语句
自定义一个类， 包含了 __enter__ 和 __exit__ 两个方法
```Python:n
def ContextManger(object):
    def __enter__(self):
        print 'Enter'
    def __exit__(self, ext_type, exc_value, traceback):
        print 'Exit'
# using Context
with ContextManger():
    print 'Hello world'
# output
>>> Enter
>>> Hello world
>>> Exit
```
+ 如果 __enter__ 后有返回值
```Python:n
def ContextManger(object):
    def __enter__(self):
        print 'Enter'
        print 'using with'
    def __exit__(self,ext_type, exc_value, traceback):
        print 'Exit'
# using Context
with ContextManger() as value:
    print value
# output
>>> Enter
>>> using with
>>> Exit
```
+ 如果处理异常
```Python:n
def ContextManger(object):
    def __enter__(self):
        print 'Enter'
    def __exit__(self, ext_type, exc_value, traceback):
        print 'Exit'
        if exc_type is not None:
            print 'value ', exc_value
with ContextManger():
    print 1 / 0
>>> Enter
>>> Exit
>>> value integer division or modulo by zero    
```
# 2 with 用例
+ 定义一个 Transaction 类
```Python:n
class Transaction(object):
    def __init__(self, connection):
        self.connection = connection
    def __enter__(self):
        return self.connection.cursor()
    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type is None:
            self.connection.commit()
            return True
        else:
            self.connection.rollback()
            return True
```
+ 使用这个类操作 sqllite 数据库
```Python:n
import sqlite3 as db
connection = db.connect(':memory:')
with Transaction(connection) as cursor:
    cursor.execute("""CREATE TABLE IF NOT EXISTS addresses (
        address_id INTEGER PRIMARY KEY,
        street_address TEXT,
        city TEXT,
        state TEXT,
        country TEXT,
        postal_code TEXT
    )""")

with Transaction(connection) as cursor:
    cursor.executemany("""INSERT OR REPLACE INTO addresses VALUES (?, ?, ?, ?, ?, ?)""", [
        (0, '515 Congress Ave', 'Austin', 'Texas', 'USA', '78701'),
        (1, '245 Park Avenue', 'New York', 'New York', 'USA', '10167'),
        (2, '21 J.J. Thompson Ave.', 'Cambridge', None, 'UK', 'CB3 0FA'),
        (3, 'Supreme Business Park', 'Hiranandani Gardens, Powai, Mumbai', 'Maharashtra', 'India', '400076'),
    ])
with Transaction(connection) as cursor:
    cursor.execute("""INSERT OR REPLACE INTO addresses VALUES (?, ?, ?, ?, ?, ?)""",
        (4, '2100 Pennsylvania Ave', 'Washington', 'DC', 'USA', '78701'),
    )
    raise Exception("out of addresses")
```

当插入第5个记录时候，引发异常，那么该事务将会 rollback. 当继续查询时候，结果如下
```Python:n
with Transaction(connection) as cursor:
    cursor.execute("SELECT * FROM addresses")
    for row in cursor:
        print row
>>>(0, u'515 Congress Ave', u'Austin', u'Texas', u'USA', u'78701')
>>>(1, u'245 Park Avenue', u'New York', u'New York', u'USA', u'10167')
>>>(2, u'21 J.J. Thompson Ave.', u'Cambridge', None, u'UK', u'CB3 0FA')
>>>(3, u'Supreme Business Park', u'Hiranandani Gardens, Powai, Mumbai', u'Maharashtra', u'India', u'400076')
```
可以发现第5条记录并没有插入到数据库中
# 3 contextlib 使用
Python 内置库提供了 Contextlib 来简化操作
```Python:n
from contextlib import contextmanager
@contextmanager
def my_contextManager():
    print 'Enter'
    yield 
    print 'Exit'

with my_contextManager():
    print 'hello world'
>>> Enter
>>> hello world!
>>> Exit
```
**yield** 关键词就是操作的内容，当然也可以进行 try 语句包含，来处理异常情况
# 4 小结
当涉及到系统资源处理时候，如文件，网络，数据库等，当发生异常时候往往会导致资源泄露或者脏数据读写。所以一般解决方案是嵌套一层 try 语句，但是大块的 try 语句影响程序阅读，所以使用with 语句，使代码能够变得漂亮。
在 C# 中用户也可以通过类实现 IDispose 借口完成同样的工作，实现借口 Dispose() 函数，而且微软也提供了一使用范例。那么类使用者可以通过 using 语句完成类的使用。