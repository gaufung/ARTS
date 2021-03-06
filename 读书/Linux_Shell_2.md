---
title: Linux Shell 阅读笔记（2)
status: public
date:  2017-10-20
---
# 1 进程
## 1.1 ps

+ `ps` 命令只列出与当前进程终端会话相关的进程
+ `ps x` 显示所有进程，进程状态列表：
    + `R`运行
    + `S`正在睡眠
    + `D`不可中断睡眠, 进程正在等待 I/O
    + `T` 已停止, 已经指示进程停止运行
    + `Z`一个死进程或“僵尸”进程
+ `ps aux` 这个选项组合，能够显示属于每个用户的进程信息，包含的信息有
    + `USER`用户
    + `%CPU`CPU使用率
    + `%MEM`内存使用率
    + `VSZ`虚拟内存大小
    + `RSS`进程占用的物理内存的大小
    + `START`进程运行的起始时间

## 1.2 top
top 程序连续显示系统进程更新的信息

## 1.3 后台执行
启动一个程序，让它立即在后台 运行，我们在程序命令之后，加上"&"字符，使用`jobs`查看后台任务。
```shell
[me@linuxbox ~]$ python &
[1] 28236
[me@linuxbox ~]$
```
使用`fg`命令，将一个后台进程调回前台，使用百分号`%[number]`表示唤醒的后台程序序号。使用`bg`将一个进程调入到后台。

## 1.4 kill
```shell
kill [-signal] PID
```
`[-signal]`的种类有：

+ `HUP`挂起
+ `INT`中断
+ `KILL`杀死
+ `TERM`终止
+ `CONT`继续
+ `STOP`停止

# 2 Shell 环境

## 2.1变量
+ Shell变量： 是bash存放的一些少量数据
+ 环境变量：其他变量

`printenv`输出环境变量； `set`命令输出Shell变量和环境变量。
## 2.2 建立Shell环境
当我们登录系统后，启动 bash 程序，并且会读取一系列称为启动文件的配置脚本，这些文件定义了默认的可供所有用户共享的 shell 环境。然后是读取更多位于我们自己主目录中 的启动文件，这些启动文件定义了用户个人的 shell 环境。精确的启动顺序依赖于要运行的 shell 会话类型。
### 登录shell
+ `/etc/profile`应用于所有用户的全局配置脚本
+ `~/.bash_profile`用户私人的启动文件，可以用来扩展或重写全局配置脚本中的设置
+ `~/.bash_login`如果文件 ~/.bash_profile 没有找到，bash 会尝试读取这个脚本

### 非登录shell
+ `/etc/bash.bashrc`应用于所有用户的全局配置文件
+ `~/.bashrc`用户私有的启动文件

# 3 Vim
## 3.1 退出
冒号`:`也是命令的一部分
```vim
:q
```
如果做了相关的修改，却没有保存，无法退出vim，使用
```vim
:q!
```

##3.2 保存工作
```vim
:w
```

## 3.3 移动光标
在vim的命令模式

+ `l`向右移动一个字符
+ `h`向左移动一个字符
+ `j`向下移动一行
+ `k`向上移动一行
+ `0`移动到当前行的行首
+ `^`移动到当前行的第一个非空字符
+ `$`移动到当前行的末尾
+ `w`移动到下一个单词或标点符号的开头
+ `W`移动到下一个单词的开头，忽略标点符号
+ `b`移动到上一个单词或标点符号的开头
+ `B`移动到上一个单词的开头，忽略标点符号
+ `ctrl+f`向下翻一页
+ `ctrl+b`向上翻一页
+ `numberG`移动到第 number 行
+ `G`移动到文件末尾

## 3.4 基本编辑
### 追加文件
使用命令`a`，光标插入改行的最后。
### 打开一行
- `o`:在当前行下方打开一行
- `O`：在当前行的上方打开一行
### 删除文本
- `x`当前字符
- `3x`当前字符及其后的两个字符
- `dd`当前行
- `5dd`当前行及随后的四行文本
- `dW`从光标位置开始到下一个单词的开头
- `d$`从光标位置开始到当前行的行尾
- `d0`从光标位置开始到当前行的行首
- `d^`从光标位置开始到文本行的第一个非空字符
- `dG`从当前行到文件的末尾
- `d20G`从当前行到文件的第20行
### 复制和粘贴文本
**复制命令**

- `yy`当前行
- `5yy`当前行及随后的四行文本
- `yW`从当前光标位置到下一个单词的开头
- `y$`从当前光标位置到当前行的末尾
- `y0`	从当前光标位置到行首
- `y^`从当前光标位置到文本行的第一个非空字符
- `yG`从当前行到文件末尾。
- `y20G`从当前行到文件末尾。

**粘贴命令**
`p`将复制的内容粘贴到下面行；`P`将复制内容到当前行。

### 连接行
大写的`J`把行与行连接起来

## 3.5 查找和替换
### 查找一行
`f[alphabet]` 该行下一个该`alphabet`的位置
### 查找文件
输入`/`命令，输入回车，开始查找，输入`n`重复查找动作。
### 全局查找和替换
```vim
:%s/Line/line/g
```
命令分解解释
- `%`表示第一行到最后一行，也可以用`1,5`表示1到5行；`1,$`表示第一行到最后一行
- `s`指定替换操作
- `/Line/line`查找和替换的文本
- `g`这是“全局”的意思

# 4 存储媒介
## 4.1 查看挂载文件系统
```shell
[me@linuxbox ~]$ mount
/dev/sda2 on / type ext3 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
/dev/sda5 on /home type ext3 (rw)
/dev/sda1 on /boot type ext3 (rw)
tmpfs on /dev/shm type tmpfs (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
fusectl on /sys/fs/fuse/connections type fusectl (rw)
/dev/sdd1 on /media/disk type vfat (rw,nosuid,nodev,noatime,
uhelper=hal,uid=500,utf8,shortname=lower)
twin4:/musicbox on /misc/musicbox type nfs4 (rw,addr=192.168.1.4)
```
### 卸载挂载
```sh
[me@linuxbox ~]$ su -
Password:
[root@linuxbox ~] umount /dev/hdc
```
### 挂载设备
```shell
[me@linuxbox ~]$ sudo mkdir /mnt/flash
[me@linuxbox ~]$ sudo mount /dev/sdb1 /mnt/flash
```

# 5 网络系统
## 5.1 检查和检测网络
**ping**
ping 命令发送一个特殊的网络数据包，叫做 IMCP ECHO_REQUEST，到 一台指定的主机。大多数接收这个包的网络设备将会回复它，来允许网络连接验证。

**traceroute**
这个 traceroute 程序（一些系统使用相似的 tracepath 程序来代替）会显示从本地到指定主机 要经过的所有“跳数”的网络流量列表。

**netstat**
netstat 程序被用来检查各种各样的网络设置和统计数据
```shell
[me@linuxbox ~]$ netstat -ie
eth0    Link encap:Ethernet HWaddr 00:1d:09:9b:99:67
        inet addr:192.168.1.2 Bcast:192.168.1.255 Mask:255.255.255.0
        inet6 addr: fe80::21d:9ff:fe9b:9967/64 Scope:Link
        UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
        RX packets:238488 errors:0 dropped:0 overruns:0 frame:0
        TX packets:403217 errors:0 dropped:0 overruns:0 carrier:0
        collisions:0 txqueuelen:100 RX bytes:153098921 (146.0 MB) TX
        bytes:261035246 (248.9 MB) Memory:fdfc0000-fdfe0000

lo      Link encap:Local Loopback
        inet addr:127.0.0.1 Mask:255.0.0.0
...
```
## 5.2  网络中传输文件
### ftp命令
- `ftp fileserver` 唤醒 ftp 程序，让它连接到 FTP 服务器，fileserver
- `anonymous`登录名,匿名登录
- `cd` 改变目录
- `ls` 列出远端文件
- `lcd Desktop`跳转到本地系统中的 ~/Desktop 目录下
- `get file`获取文件至本地
- `bye` 退出

### wget
`wget http://linuxcommand.org/index.php`

## 5.3 与远程主机安全通信
SSH 解决了这两个基本的和远端主机安全交流的问题