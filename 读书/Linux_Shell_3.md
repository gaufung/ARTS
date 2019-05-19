---
title: Linux Shell 阅读笔记（3)
date: 2017-10-03
status: public
---

# 1 查找文件
## 1.1 locate
这个 locate 程序快速搜索路径名数据库，并且输出每个与给定字符串相匹配的文件名.
```shell
[me@linuxbox ~]$ locate bin/zip
```
## 1.2 find
### 输出列表
```shell
find [path]
```
### 测试
*只列出目录列表*
```shell
find ~ -type d
```
`find`支持的命令有
- `b` 块设备文件
- `c`字符设备文件
- `d`目录
- `f`普通文件
- `l`连接符号

*文件名通配符和大小*
```shell
[me@linuxbox ~]$ find ~ -type f -name "\*.JPG" -size +1M | wc -l
```
- `b`512 个字节块
- `c`字节
- `w`两个字节的字
- `k`千字节
- `M`兆字节
- `G`千兆字节

*测试条件*
- `-cmin n`匹配的文件和目录的内容或属性最后修改时间正好在 n 分钟之前。 指定少于 n 分钟之前，使用 -n，指定多于 n 分钟之前，使用 +n
- `-cnewer file`匹配的文件和目录的内容或属性最后修改时间早于那些文件。
- `-ctime n`匹配的文件和目录的内容和属性最后修改时间在 n*24小时之前。
- `-empty` 匹配空文件和目录
- `-group name`匹配的文件和目录属于一个组
- `-iname pattern`就像-name 测试条件，但是大小写敏感。
- `-inum -n`匹配的文件的 inode 号是 n
- `-mmin n`匹配的文件或目录的内容被修改于 n 分钟之前。
- `-mtime n`匹配的文件或目录的内容被修改于 n*24小时之前。
- `-name pattern`用指定的通配符模式匹配的文件和目录
- `-newer file`匹配的文件和目录的内容早于指定的文件
- `-nouser`匹配的文件和目录不属于一个有效用户
- `-nogroup`匹配的文件和目录不属于一个有效的组
- `-perm mode`匹配的文件和目录的权限已经设置为指定的 mode。mode 可以用 八进制或符号表示法。
- `-samefile name`匹配和文件 name 享有同样 inode 号的文件
- `-size n`匹配的文件大小为 n
- `-type c`匹配的文件类型是 c
- `-user name`匹配的文件或目录属于某个用户

### 操作符
多种测试条件结合起来
- `-and`匹配如果操作符两边的测试条件都是真， 简写-a
- `-or`匹配若操作符两边的任一个测试条件为真， 简写-o
- `-not`匹配若操作符后面的测试条件是真， 简写！
- `()`把测试条件和操作符组合起来形成更大的表达式

### 预定义操作
- `-delete`删除当前匹配的文件
- `-ls`对匹配的文件执行等同的 ls -dils 命令
- `-print`把匹配文件的全路径名输送到标准输出
- `-quit`一旦找到一个匹配

### xargs
它从标准输入接受输入，并把输入转换为一个特定命令的 参数列表
```shell
find ~ -type f -name 'foo\*' -print | xargs ls -l
```

# 2 归档和备份

## 2.1 gzip和gunzip
```shell
[me@linuxbox ~]$ ls -l /etc > foo.txt
[me@linuxbox ~]$ ls -l foo.*
-rw-r--r-- 1 me     me 15738 2008-10-14 07:15 foo.txt
[me@linuxbox ~]$ gzip foo.txt
[me@linuxbox ~]$ ls -l foo.*
-rw-r--r-- 1 me     me 3230 2008-10-14 07:15 foo.txt.gz
[me@linuxbox ~]$ gunzip foo.txt
[me@linuxbox ~]$ ls -l foo.*
-rw-r--r-- 1 me     me 15738 2008-10-14 07:15 foo.txt
```

`gzip`还有其他选项
- `-c`把输出写入到标准输出，并且保留原始文件
- `-d`解压缩，正如 gunzip 命令一样
- `-f`	强制压缩，即使原始文件的压缩文件已经存在了，也要执行
- `-h` help
- `-l`列出每个被压缩文件的压缩数据
- `-r`若命令的一个或多个参数是目录，则递归地压缩目录中的文件
- `-t`测试压缩文件的完整性
- `-v`verbose
- `-number`设置压缩指数。number 是一个在1（最快，最小压缩）到9（最慢，最大压缩）之间的整数。 数值1和9也可以各自用--fast 和--best 选项来表示。默认值是整数6。

## 2.2 tar
我们经常看到扩展名为.tar 或者.tgz 的文件，它们各自表示“普通” 的 tar 包和被 gzip 程序压缩过的 tar 包。
**模式**
- `c`为文件和／或目录列表创建归档文件
- `x`抽取归档文件
- `r`追加具体的路径到归档文件的末尾
- `t`列出归档文件的内容

```shell
tar cf playground.tar playground
tar cf playground2.tar ~/playground
```

## 2.3 zip
`zip options zipfile file...`

## 2.4 同步文件和目录
使用 rsync 远端更新协议，此协议允许 rsync 快速地检测两个目录的差异，执行最小量的复制来达到目录间的同步。比起其它种类的复制程序， 这就使 rsync 命令非常快速和高效。
```shell
rsync -[options] source destination
```
这里 source 和 destination 是下列选项之一:
- 一个本地文件或目录
- 一个远端文件或目录，以[user@]host:path 的形式存在
- 一个远端 rsync 服务器，由 rsync://[user@]host[:port]/path 指定

# 3 正则表达式
## 3.1 grep
grep接受的选项和参数
```shell
grep [options] regex [file...]
```
常用的`grep`的选项有
- `-i` 忽略大小写
- `-v`输出不匹配的选项
- `-c`打印匹配的数量
- `-l`打印包含匹配项的文件名，而不是文本行本身
- `-L`相似于-l 选项，但是只是打印不包含匹配项的文件名
- `-n`	在每个匹配行之前打印出其位于文件中的相应行号
- `-h`应用于多文件搜索，不输出文件名

##3.2 POSIX 字符集
- `[:alnum:]`字母数字字符。在 ASCII 中，等价于：[A-Za-z0-9]
- `[:word:]`与[:alnum:]相同, 但增加了下划线字符。
- `[:alpha:]`字母字符。在 ASCII 中，等价于：[A-Za-z]
- `[:blank:]`包含空格和 tab 字符。
- `[:cntrl:]`ASCII 的控制码。包含了0到31，和127的 ASCII 字符。
- `[:digit:]`数字0到9
- `[:graph:]`可视字符。在 ASCII 中，它包含33到126的字符
- `[:lower:]`小写字母。
- `[:punct:]`标点符号字符
- `[:print:]`可打印的字符
- `[:space:]`空白字符，包括空格，tab，回车，换行，vertical tab, 和 form feed.在 ASCII 中， 等价于：[ \t\r\n\v\f]
- `[:upper:]`大写字母
- `[:xdigit:]`用来表示十六进制数字的字符

# 4 编译
```shell
./configure
make
make install
```