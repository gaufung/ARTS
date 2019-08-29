---
date: 2018-03-20
status: draft
tags: Algorithms
title: Gojson 解析器实现
---

[gojson](https://github.com/gaufung/gojson)
# 背景
[JSON](https://en.wikipedia.org/wiki/JSON) 是Web中常用的数据交换的格式，相对于 [XML](https://en.wikipedia.org/wiki/XML) 而言，JSON具有可阅读性的特点，而且每个 JSON 对象都能与编程语言中的对象对应起来，通常来讲为一个字典对象。 一个典型的JSON数据如下所示
```JSON
{
  "firstName": "John",
  "lastName": "Smith",
  "isAlive": true,
  "age": 27,
  "address": {
    "streetAddress": "21 2nd Street",
    "city": "New York",
    "state": "NY",
    "postalCode": "10021-3100"
  },
  "phoneNumbers": [
    {
      "type": "home",
      "number": "212 555-1234"
    },
    {
      "type": "office",
      "number": "646 555-4567"
    },
    {
      "type": "mobile",
      "number": "123 456-7890"
    }
  ],
  "children": [],
  "spouse": null
}
```
那么相对于 `Go`语言的中就是一个 `Map` 字典，其中类型为`map[string]interface{}`，其中 `Key` 为`String` 类型， `Value`值为 `interface{}`。因为每一个 JSON 对象的值并不是同一种类型，因此在使用过程中要强制转换， 那么 JSON 中 `Value` 类型主要有：
- String
- Boolean
-   Number
- Null
- Array
- Object
其中 `Array` 中的数据类型可以是其中任意一种，而 `Object`也是  JSON 对象。那么一个完整的 JSON 的推导式为
```
0. S -> Object
1. Object -> { Items }
2. Items ->    Item
                    | Item, Items
                    |
3. Item -> Key : Value
4. Key -> String
5. Value ->   String
            | Number
            | Null
            | Boolean
            | Array
            | Object

6. STRING -> "Alphabets"
7. Alphabets ->  Alphabet
                | Alphabet Alphabets
                |
8. Number -> digits
9. Null -> null
10. Boolean ->   true
               | false
11. Array -> [ Elements ]
12. Elements ->   Value
                | Value, Elements
                |
```
#  Go 语言实现JSON解析器
## 1 Byte Reader
在 `Go` 语言中，有两个表示字符的数据类型，分别为 `byte` 和 `rune`。 通过查看定义源码:
```go
type byte = uint8
type rune = int32
```
`byte`表示为一个字节，而 `rune` 为四个字节。 通过字符编码的相关知识可以知道， 为了保证解析器通用性，选择 `rune` 为单个字符的表示。而在包 `io` 中定义的
 `io.Reader`的接口中，只有`Read([]byte)` 函数，那么如何将`byte` 向 `rune` 转换呢？借助 `unicode/utf-8`可以方便转换
```Go
func (r *CharReader) next() rune {
	bytes := make([]byte, 0)
	for {
		bytes = append(bytes, r.nextByte())
		if utf8.Valid(bytes) {
			r, _ := utf8.DecodeRune(bytes)
			return r
		}
	}
}
```
由 *UTF-8** 编码知道，一个字符一般有1、2、3或者4个字节组成， 通过 `utf8.Valid()` 方法验证该自己数组是否有效。如果有效，通过`utf8.DecodeRune()` 方法将其转换为一个 `rune` 表示的一个字符。
## 2 Token Reader
从推导式中可以知道，JSON 中的 Token 主要有
``` go
type Token int
const (
	END_DOCUMENT = iota // 0
	START_OBJECT // 1
	END_OBJECT // 2
	START_ARRAY // 3
	END_ARRAY // 4
	COLON_SEPERATOR // 5
	COMA_SEPERATOR // 6
	STRING // 7
	BOOLEAN // 8
	NUMBER // 9
	NULL // 10
)
```
其中`END_DOCUMENT`为json文档结束, `START_OBJECT` 和 `END_OBJECT` 分别对应于`{`和`}`； `START_ARRAY`和`END_ARRAY` 分别对应于`[`和`]`。
### 2.1 跳过空白字符串
空白字符串在JSON中是没有意义的，因此在读取的时候跳过
```go
ch := '?'
for {
		if  !t.reader.hasMore() {
			return END_DOCUMENT, nil
		}
		ch = t.reader.peek() //peek it
		if !t.isWhiteSpace(ch) {
			break
		}
		t.reader.next()
}
```
### 2.2 读取数值
JSON 中数值表示形式主要有:
```
12
12.3
-12
-12.4
12E4
12E-4
-12E-4
...
```
表示为`[\+-]+[0-9]*[\.]+[0-9]*[Ee]+[\+-]+[0-9]*`，通常来讲，主要分为三个部分 `IntPart`, `FraPart` 和 `ExpPart`。加上符号，数值中包含的Token有:
```Go
READ_NUMBER_PLUS = 0x0001
READ_NUMBER_MINUS = 0x0002
READ_NUMBER_DIGIT = 0X0004
READ_NUMBER_DOT = 0x0008
READ_NUMBER_E = 0x0010
```
通过`status`变量保存解析各个阶段期待的token. 以解析`READ_NUMBER_DIGIT` 为例具体的过程如下
```go
case READ_NUMBER_DIGIT:
			if phase == READ_NUMBER_INT_PART {
				if status & READ_NUMBER_DIGIT > 0 {
					char := t.reader.next()
					intPart = append(intPart, char)
					status = READ_NUMBER_DIGIT | READ_NUMBER_DOT | READ_NUMBER_E
					continue
				}
			}
			if phase == READ_NUMBER_FRA_PART {
				if status & READ_NUMBER_DIGIT > 0 {
					char := t.reader.next()
					fraPart = append(fraPart, char)
					status = READ_NUMBER_DIGIT | READ_NUMBER_E
					continue
				}
			}
			if phase == READ_NUMBER_EXP_PART {
				if status & READ_NUMBER_DIGIT > 0 {
					char := t.reader.next()
					expPart = append(expPart, char)
					status = READ_NUMBER_DIGIT
					continue
				}
			}
			return 0.0, t.errors("Read float64: unexpect char ")
```

**To be continued**