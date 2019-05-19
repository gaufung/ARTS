---
title: Linux Shell 阅读笔记（1）
date:  2017-09-30
status: public
---
Linux Shell 学习笔记，[快乐的命令行](https://billie66.gitbooks.io/tlcl-cn/index.html)。

# 1 文件跳转

+ `cd -`跳转到上一步目录
+ `cd ~user_name`跳转到user_name的主目录

# 2 Linux文件目录

+ `/`根目录
+ `/bin`系统启动和运行的目录
+ `/boot`包含Linux内核
+ `/dev`包含设备结点的特殊目录
+ `/etc`这个目录包含所有系统层面的配置文件
+ `/home`给每个用户分配一个目录
+ `/lib`包含核心系统程序所需的库文件
+ `/lost+found`当部分恢复一个损坏的文件系统时，会用到这个目录
+ `/media`包含可移除媒体设备的挂载点
+ `/root`root用户目录
+ `/tmp`存储由各种程序创建的临时文件的地方
+ `/usr`, `/usr/bin`, `/usr/lib`, `/usr/local` and  `/usr/sbin`和普通用户所需要的所有程序和文件
+ `/var` 需要改动的文件存储的地方

# 3 文件和目录

## 3.1 通配符

- `*` 任意多个字符
- `?`任意一个字符
- `[characters]`任意一个属于字符集中的字符
- `[!characters]`任意一个不是字符集中的字符
- `[[:class:]]`任意一个属于指定字符类中的字符
    - `[:alnum:]`任意一个字母或数字
    - `[:alpha:]` 任意一个字母
    - `[:digit:]`任意一个数字
    - `[:lower:]`任意一个小写字母
    - `[:upper:]`任意一个大写字母


## 3.2 创建链接

### 硬链接

```shell
ln file link
```
硬链接是最初 Unix 创建链接的方式,  在默认情况下，每个文件有一个硬链接，这个硬链接给文件起名字。当我们创建一个 硬链接以后，就为文件创建了一个额外的目录条目。

### 符号链接
```Shell
ln -s file link
```
符号链接生效，是通过创建一个 特殊类型的文件，这个文件包含一个关联文件或目录的文本指针,删除一个符号链接时，只有这个链接被删除，而不是文件自身。如果删除这个文件 早于文件的符号链接，这个链接仍然存在，但是不指向任何东西。在这种情况下， 这个链接被称为坏链接。在许多实现中，ls 命令会以不同的颜色展示坏链接，比如说 红色，来显示它们的存在。

# 4 使用命令
命令的四种形式

1. 可执行程序(编程语言实现的程序)
2. 内置命令（builtin)
3.  Shell函数
4. 命令的别名或者组合

## 命令识别
```shell
type command
```
## 可执行命令位置
```shell
which command
```
## 帮助文档
```shell
man command
```
## 创建别名
命令组合
```shell
command1, command2, command3...
```
创建命令别名
```shell
alias new_command="command1, command2, command3..."
```
取消命令别名
```shell
unalias new_command
```

# 5 重定向

## 5.1 标准输入、输出和错误
+ `stdin`标准输入的设备（键盘）
+ `stdout`结果输送到标准输出的文件中（屏幕）
+ `stderr`状态信息输送到标准错误文件中（屏幕）

shell 内部将标准输入、标准输出和标准错误的文件描述符0，1和2。

### 5.1.1 重定向标准输出

使用`">"`命令重定向标准输出，使用`">>"`追加重定向输出。
### 5.1.2 重定向标准错误
使用`"2>"`重定向标准输出。
### 5.1.3 重定向标准输入
`cat`命令不带有参数，将会从标准输入（键盘）中读取数据
```shell
cat > foo.txt
```
## 5.2 管道
使用管道操作符 `"|"`，一个命令的标准输出可以管道到另一个命令的标准输入
```shell
command1 | command2
```

## 5.3 试用命令

+ `sort` 对结果排序
+ `uniq`报到和排除重复行, 一般在`sort`之后有效
+ `wc` 打印行，字和字节数
+ `grep`打印匹配行
+ `tee` 从 Stdin 读取数据，并同时输出到 Stdout 和文件

# 6 Shell中字符

### 字符展开

```Shell
[me@linuxbox ~]$ echo this is a test
this is a test
[me@linuxbox ~]$ echo [[:upper:]]*
Desktop Documents Music Pictures Public Templates Videos
```

### 波浪线展开

波浪线字符("~")有特殊的意思。当它用在一个单词的开头时，它会展开成指定用户的主目录名
```shell
[me@linuxbox ~]$ echo ~
/home/me
```

### 算术表达式展开

算术表达式展开的形式`$((expression))`，算术表达式只支持整数（全部是数字，不带小数点），但是能执行很多不同的操作。
### 花括号展开
**实例1**
```shell
[me@linuxbox ~]$ echo Front-{A,B,C}-Back
Front-A-Back Front-B-Back Front-C-Back
```

**实例2**
```shell
[me@linuxbox ~]$ echo Number_{1..5}
Number_1  Number_2  Number_3  Number_4  Number_5
```
**实例3**
```shell
[me@linuxbox ~]$ echo a{A{1,2},B{3,4}}b
aA1b aA2b aB3b aB4b
```

### 命令展开

命令替换允许我们把一个命令的输出作为一个展开模式来使用
```shell
[me@linuxbox ~]$ ls -l $(which cp)
-rwxr-xr-x 1 root root 71516 2007-12-05 08:58 /bin/cp
```

### 引用

**双引号**
如果把文本放在双引号中，除了`$`，`\`和`'`之外，其余的失去特殊的含义。
```shell
[me@linuxbox ~]$ echo this is a   test
this is a test
[me@linuxbox ~]$ echo "this is a    test"
this is a    test
```
**单引号**
禁止所有的引用展开
```shell
[me@linuxbox ~]$ echo "text ~/*.txt {a,b} $(echo foo) $((2+2)) $USER"
text ~/*.txt   {a,b} foo 4 me
[me@linuxbox ~]$ echo 'text ~/*.txt {a,b} $(echo foo) $((2+2)) $USER'
text ~/*.txt  {a,b} $(echo foo) $((2+2)) $USER
```
**转义字符**
有时候我们只想引用单个字符。我们可以在字符之前加上一个反斜杠，在这个上下文中叫做转义字符。
+ `\a`响铃
+ `\b`退格符
+ `\n`新的一行。在类似 Unix 系统中，产生换行
+ `\r`回车符
+ `\t`制表符

# 7 键盘技巧

## 7.1 命令行技巧

### 移动光标

+ `ctrl+a`移动光标到行首
+ `ctrl+e`移动光标到行尾
+ `Ctrl-f`光标前移一个字符
+ `ctrl+b`光标后移一个字符
+ `alt-f`光标向前移动一个单词
+ `alt-b`光标向后移动一个单词
+ `ctrl-l`清空屏幕

对于Mac OS用户，没有`alt`键，[参考该方法](http://www.cnblogs.com/noblog/p/alt-key-in-osx-terminal.html)

### 修改文本

+ `ctrl-d`删除光标位置的字符。
+ `ctrl-t`光标位置的字符和光标前面的字符互换位置
+ `alt-t`光标位置的字和其前面的字互换位置。
+ `alt-l`把从光标位置到字尾的字符转换成小写字母
+ `alt-u`把从光标位置到字尾的字符转换成大写字母。

### 剪切和粘贴文本

+ `ctrl-k`剪切从光标位置到行尾的文本。
+ `ctrl-u`剪切从光标位置到行首的文本。
+ `alt-d`剪切从光标位置到词尾的文本。
+ `alt-backspace`剪切从光标位置到词头的文本。如果光标在一个单词的开头，剪切前一个单词。
+ `ctrl-y`把剪切环中的文本粘贴到光标位置。

# 8 权限

## 8.1拥有者，组成员，和其他人

使用`id`命令，查询自己相关命令
```shell
[me@linuxbox ~]$ id
uid=500(me) gid=500(me) groups=500(me)
```

## 8.2读取，写入，和执行

**文件类型**

+  `-`普通文件
+ `d`一个目录
+ `l`符号链接
+ `c`字符链接
+ `b`设备文件

**文件模式**

+ `r`可读
+ `w`可写
+ `x`可执行

**改变文件模式**

### 数字表示法
```shell
[me@linuxbox ~]$ ls -l foo.txt
-rw-rw-r-- 1 me    me    0  2008-03-06 14:52 foo.txt
[me@linuxbox ~]$ chmod 600 foo.txt
[me@linuxbox ~]$ ls -l foo.txt
-rw------- 1 me    me    0  2008-03-06 14:52 foo.txt
```

### 符号表示法
- `u` user的简写，意思是文件或目录的所有者。
- `g` 用户组 
- `o` others，意思是其他所有的人
- `a` all，是“u”，“g”，和 “o” 三者的联合。
chmod 符号表示法
- `u+x`:为文件所有者添加可执行权限。
- `u-x`删除文件所有者的可执行权限。
- `+x`为文件所有者，用户组，和其他所有人添加可执行权限。等价于 a+x。
- `o-rw`删除其他人的读权限和写权限。
- `go=rw`给群组的主人和任意文件拥有者的人读写权限。如果群组的主人或全局之前已经有了执行的权限，他们将被移除。
- `u+x,go=rw`给文件拥有者执行权限并给组和其他人读和执行的权限。多种设定可以用逗号分开。

## 8.3 更改身份

### su
su 命令用来以另一个用户的身份来启动 shell
```shell
su [-[l]] [user]
```
如果不指定用户，那么就假定是 超级用户。注意（不可思议地），选项"-l"可以缩写为"-"，这是经常用到的形式
### sudo
以另一个用户身份执行命令

## 8.4 chown－更改文件所有者和用户组
chown 命令被用来更改文件或目录的所有者和用户组。使用这个命令需要超级用户权限。chown 命令 的语法看起来像这样：
```shell
chown [owner][:[group]] file...
```
+ `bob`把文件所有者从当前属主更改为用户 bob。
+ `bob:users`把文件所有者改为用户 bob，文件用户组改为用户组 users。
+ `:users`把文件用户组改为组 admins，文件所有者不变。
+ `bob:`文件所有者改为用户 bob，文件用户组改为，用户 bob 登录系统时，所属的用户组。

## 8.5更改密码

```shell
passwd [user]
```