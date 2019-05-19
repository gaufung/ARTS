---
date: 2017-01-25 13:46
status: public
title: Python字符串编码
---

# 0 基本概念
1. 字符(character)
字符是一个信息单位，它是各种文字和符号的总称。
2. 字符集（Character set)
字符集是字符的集合。字符集的种类较多，每个字符集包含的字符个数也不同。比如，常见的字符集有 ASCII 字符集、GB2312 字符集、Unicode 字符集等。
3. 字符编码（Character encoding）
是指对于字符集中的字符，将其编码为特定的二进制数，以便计算机处理。常见的字符编码有 ASCII 编码，UTF-8 编码，GBK 编码等。

# 1 ASCII 字符编码和 Unicode 字符编码
## 1.1 ASCII 字符编码
英文字母，数字和一些普通符号跟二进制的转换关系，被称为 ASCII (American Standard Code for Information Interchange，美国信息互换标准编码) 码。

比如，大写英文字母 A 的二进制表示是 01000001（十进制 65），小写英文字母 a 的二进制表示是 01100001 （十进制 97），空格 SPACE 的二进制表示是 00100000（十进制 32），不过ASCII只用到了其中的一半（\x80以下）。

## 1.2 Unicode 字符编码
ASCII 码只规定了 128 个字符的编码，这在美国是够用的。可是，计算机后来传到了欧洲，亚洲，乃至世界各地，而世界各国的语言几乎是完全不一样的，用 ASCII 码来表示其他语言是远远不够的。
**解决方案**：所有语言的字符都用同一种字符集来表示，这就是Unicode。
最初的Unicode标准UCS-2使用两个字节表示一个字符，所以通常有人说Unicode使用两个字节表示一个字符的说法。但过了不久有人觉得256*256太少了，还是不够用，于是出现了UCS-4标准，它使用4个字节表示一个字符，不过最多的仍然是UCS-2。
UCS(Unicode Character Set)还仅仅是字符对应码位的一张表而已，比如"汉"这个字的码位是6C49。字符具体如何传输和储存则是由UTF(UCS Transformation Format)来负责。
开始直接使用UCS的码位来保存，这就是UTF-16，比如，"汉"直接使用\x6C\x49保存(UTF-16-BE)，或是倒过来使用\x49\x6C保存(UTF-16-LE)。但以前英文字母只需要一个字节就能保存了，现在变成了两个字节，空间消耗大了一倍，于是 **UTF-8** 编码方案提出。
UTF-8 (8-bit Unicode Transformation Format) 是一种针对 Unicode 的可变长度字符编码，它使用一到四个字节来表示字符，例如，ASCII 字符继续使用一个字节编码，阿拉伯文、希腊文等使用两个字节编码，常用汉字使用三个字节编码，等等。

# 2 Python 2 中字符串
在 Python 2.x 中默认是的编码方式为 ascii，而在 Python 3.x使用的为 unicode。
```Python
>>>import sys
>>>sys.getdefaultencoding()
```

如果想在python 2.x 中使用 unicode 编码方式，那么文件中的添加 `from __futue__ import unicode_literals` 那么将会使用 Unicode 编码方式，而不是 Ascii 编码方式。

`Python 2.x` 中有两种和字符串相关的类型：`str` 和 `unicode`，它们的父类是 `basestring`。其中，`str` 类型的字符串有多种编码方式，默认是 `ascii`，还有 `gbk`，`utf-8` 等，`unicode` 类型的字符串使用 u'...' 的形式来表示，下面的图展示了 `str` 和 `unicode` 之间的关系：

![](~/14-50-04.png)

```python
u = u'汉'
print repr(u) # u'\u6c49'
s = u.encode('UTF-8')
print repr(s) # '\xe6\xb1\x89'
u2 = s.decode('UTF-8')
print repr(u2) # u'\u6c49
```
# 3 字符串异常
用 Python2 编写程序的时候经常会遇到 `UnicodeEncodeError` 和 `UnicodeDecodeError`，它们出现的根源就是如果代码里面混合使用了 `str` 类型和 `unicode` 类型的字符串，Python 会默认使用 `ascii` 编码尝试对 `unicode` 类型的字符串编码 (`encode`)，或对 `str` 类型的字符串解码 (`decode`)，这时就很可能出现上述错误。
## 3.1 UnicodeEncodeError
如果函数或类等对象接收的是 str 类型的字符串，但你传的是 unicode，Python2 会默认使用 ascii 将其编码成 str 类型再运算，这时就很容易出现 UnicodeEncodeError。
```python
>>> u_str = u'你好'
>>> str(u_str)
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-1: ordinal not in range(128)
```

## 3.2 UnicodeDecodeError
在进行同时包含 str 类型和 unicode 类型的字符串操作时，Python2 一律都把 str 解码（decode）成 unicode 再运算，这时就很容易出现 UnicodeDecodeError。
```python
>>> s = '你好'    # str 类型, utf-8 编码
>>> u = u'世界'   # unicode 类型
>>> s + u        # 会进行隐式转换，即 s.decode('ascii') + u
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0: ordinal not in range(128)
```

为了避免出错，我们就需要显示指定使用 ‘utf-8’ 进行解码，如下：

```python
>>> s = '你好'                 # str 类型，utf-8 编码
>>> u = u'世界'
>>> s.decode('utf-8') + u     # 显示指定 'utf-8' 进行转换
u'\u4f60\u597d\u4e16\u754c'   # 注意这不是错误，这是 unicode 字符串
```

# 4 python 文件读写
内置的open()方法打开文件时，read()读取的是`str`，读取后需要使用正确的编码格式进行decode()。write()写入时，如果参数是`unicode`，则需要使用你希望写入的编码进行encode()，如果是其他编码格式的`str`，则需要先用该`str`的编码进行decode()，转成`unicode`后再使用写入的编码进行encode()。如果直接将`unicode`作为参数传入write()方法，Python将先使用源代码文件声明的字符编码进行编码然后写入。
```python
f = open('test.txt')
s = f.read()
f.close()
print type(s) # <type 'str'>
# 已知是GBK编码，解码成unicode
u = s.decode('GBK')
f = open('test.txt', 'w')
# 编码成UTF-8编码的str
s = u.encode('UTF-8')
f.write(s)
f.close()
```