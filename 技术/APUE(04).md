---
date: 2019-06-19
status: public
tags: APUE
title: APUE(04)
---

# 1 `stat、fstat、fstatat` 和 `lstat` 函数
函数原型 
```c
# include <sys/stat.h>
int stat(const char *restrict pathname, sturct stat *restrict buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);
```
- stat 返回此命名文件的信息结构
- fstat 返回文件描述符 `fd` 上打开的文件有关信息
- lstat 返回链接文件的信息，而不是该符号链接的文件的信息
- fstatat 返回当前打开的目录（有 `fd` 指向） 的路径名文件统计信息

文件信息结构 
```c
struct stat{
  mode_t st_mode; /*file type & mode (permission) */  
  ino_t st_ino;  /*i-node number (serial number)*/
  dev_t st_dev; /*device number (file system)*/
  dev_t st_rdev; /*device number for special files*/
  nlink_t st_nlink; /*number of links*/
  uid_t st_uid; /*user ID of the owner*/
  gid_t st_gid; /*group ID of the owner*/
  off_t st_size; /*sieze in bytes,  for regular files */
  struct timespec st_atime; /*time of last access */
  struct timespec st_mtime; /* time of last modification */
  struct timespec st_ctime; /* time of last file status change */
  blksize_t st_blksize; /*best I/O block size */
  blkcnt_t st_blocks; /*number of disk blocks allocated */
};
```

# 2 文件类型
1. 普通文件 (`regualr file`)
2. 目录文件（`directory file`)
3. 块特殊文件（`block special file`) 提供对设备带缓冲的访问;
4. 字符特殊文件（`character special file`)提供设备不带缓冲的访问；
5. `FIFO` 用于进程之间之间的通信，又称为命令管道；
6. 套接字（`socket`) 用于进程之间网络通信；
7. 符号链接（`symbolic link`) 文件指向另一个文件；

文件类型信息包含在 `st_mode` 字段中。可以通过相关的宏确定文件类型。

# 3 用户 `ID` 和组 `ID`
一个进程关联的 `ID` 如下
- 我们实际上是谁（从登陆口令文件中获取）
    - 实际用户 ID
    - 实际组 ID
- 用于文件访问权限检查（通常与实际用户 ID 和 实际组 ID 相同）
    - 有效用户 ID 
    - 有效组 ID
    - 附属组 ID 
- 由 `exec` 函数保存（执行时候包含有效用户 ID 和有效组 ID 的副本）
    - 保存的设置用户 ID 
    - 保存的设置组 ID 

通常有效用户 ID 等于实际用户 ID， 有效组 ID 等于实际组 ID。在 `st_mode` 中设置一个特殊标志，当执行这个文件时候，进程有效用户 ID 为文件所有者 ID， 有效组等于文件所有组 ID。

# 4 文件访问权限
每个文件都有 9 个访问位权限，可以分为3类
- 用户相关
    - `S_IRUSR` 用户读
    - `S_IWUSR` 用户写
    - `S_IXUSR` 用户执行
- 组相关
    - `S_IRGRP` 组读
    - `S_IWGRP` 组写
    - `S_IXGRP` 组执行
- 其他相关
    - `S_IROTH` 其他读
    - `S_IWOTH` 其他写
    - `S_IXOTH` 其他执行

可以使用 `chmod` 可以修改这 9 个权限， `u` 表示用户（所有者），`g` 表示组，`o` 表示其他。补充说明
1. 如果用名字打开任一类型的文件，该名字中包含的每一个目录都是可执行的。读权限只能获得该目录中文件名列表；
2. 对文件的读写权限决定了是否能够打开现有文件读写操作；
3. `open` 函数中指定 `O_TRUNC` 标志，必须要求对文件有写权限；
4. 在目录总创建新文件，必须对目录有写和执行权限；
5. 删除一个文件，必须要改文件的目录写和执行权限，而不管文件本身读写权限；
6. `exec` 函数执行任何一个文件，必须这个文件有执行权限（必须是普通文件）

进程每次打开、创建和删除文件的时候，内核需要进行如下测试
- 若进程有效用户 ID 是 `0` (超级用户)，则允许访问；
- 如果进程用户 ID 等于文件所有者 ID，则按照文件的所有者访问权限设置；
- 如果进程有效组 ID等于文件组 ID， 则按照文件所有组访问权限设置；
- 如果进程是其他用户，按照文件其他用户访问权限设置。
上述步骤只能从上到下，并且只能选择其中的一个。

新建文件的用户 ID 为进程有效用户 ID，组 ID 为下面选项中的一个：
1. 进程有效组 ID 
2. 所在目录的组 ID 

# 5 `access` 和 `faccestat` 函数
函数原型 
```c
# include <unistd.h>
int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);
```
使用 `access` 和 `faccessat` 函数按照实际用户 ID 和实际组 ID 对访问权限进行测试。

mode | 说明
:---: | :---:
`R_OK` | 测试读权限
`W_OK` | 测试写权限
`X_OK` | 测试执行权限

# 6 `umask` 函数
函数原型 
```c
# include <sys/stat.h>
mode_t umask(mode_t cmask)
```
创建屏蔽字中 `1` 的位，在文件 `mode` 中相应的位置一定被关闭。

# 7 `chmod`, `fchmod` 和 `fchmodat`
函数原型 
```c
# include <sys/stat.h>
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
```
这三个函数可以帮助我们修改现有文件的访问权限。
更改文件的的权限位，进程有效用户 ID 必须等于文件所有者的 ID，或者进程必须具有超级用户的权限。

# 8 粘着位
如果对一个目录设置了粘着位，那么对该目录具有写权限的用户并且满足下面条件之一的才能删除或者重命名目录下的文件：
- 拥有此文件
- 拥有此目录
- 是超级用户
使用 `ll` 命令查看目录其它用户权限是 `t` 而不是 `[r|w|x]`。
# 9 `chown`, `fchown`, `fchownat` 和 `lchown` 函数
函数原型
```c
#include <unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);
```
可用于修改文件用户 ID 和组 ID，`owner` 和 `group` 任意参数为 `-1`, 则原先所属不变。基于 `BSD` 的系统一直规定只有超级用户可以更改文件的所有者，而 `System V` 则规定任意用户更改拥有者的文件的所有者。

# 10 文件长度
`st_size` 字段支队普通文件、目录文件和符号链接有意义。
- 普通文件为文件大小；
- 目录文件为一个数的整数倍；
- 链接文件是*文件名*的实际字节数；

# 11 文件截断
函数原型 
```c
# include <unistd.h>
int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);
```
将现有文件截断为长度 `length`，如果文件以前大于 `length`, 则 `length` 之外的数据不再被访问；如果之前文件小于 `length`，则文件长度将增加，中间填充 `0` (形成空洞)

# 12 文件系统

![](./_image/2019-06-24-06-18-23.jpg?r=63)
将一个磁盘分成多个分区，每个分区可以包含一个文件系统，节点的是固定长度的记录项，包含文件的大部分信息。
在下图的实例中

![](./_image/2019-06-24-06-35-33.jpg?r=62)
- 有两个目录项指向同一个 i 节点，每个 i 节点中都有一个链接计数，其值为指向 i 节点的目录项数。只有链接计数减少到 0， 才可删除文件（称为硬链接）。
- 符号链接的实际内容包含了该符号链接所指向文件的名字。
- i 节点包含了文件的全部信息：文件类型、文件访问权限、文件长度和指向文件的指针。
- 目录项中的 i 节点编号指向同一文件系统中相应的 i 节点。
- 在同一文件系统中，重命名文件时候，只是重新构建指向 i 节点的目录项，并删除老的目录项。

那么目录的文件系统形式是这样的呢？
```shell
$ mkdir testdir
```

![](./_image/2019-06-24-07-20-45.jpg?r=61)
编号 `2549` 的 i 节点，链接数为 2， 来自命名目录 `testdir` 和该目录中的 `.` 项。编号 `1267` 的 i 节点，其类型为一个目录，至少有 3 个链接指向它。在父目录中每一个子目录都会是该父目录的链接计数加一。

# 13 `link`, `linkat`, `unlink`,`unlinkat` 和 `remove`
函数原型
```c
# incldue <unistd.h>
int link(const char *exsitingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);
```
创建一个新的目录项 `newpath`，它引用现有文件 `exsitingpath`。
函数原型：
```c
# include <unistd.h>
int unlink(const char *pahtname);
int unlinkat(int fd, const char *pathname, int flag);
```
删除这个目录项，并且将 `pathname` 有引用的链接计数减 1。

# 14 `rename` 和 `renameat` 函数
函数原型
```c
# include <stdio.h>
int rename(const char *oldname, const char *newname);
int renameat(int oldfd, const char *oldname , int newfd, const char *newname);
```
1. 如果 `oldname` 为一个文件而不是目录，则为该文件或者符号链接重命名。如果 `newname` 已经存在，那么不能引用一个目录，如果 `newname` 存在且不是一个目录，那么将目录项删去然后将 `oldname` 重命名为 `newname`。
2. 如果 `oldname` 是一个目录，则为目录重命名。如果  `newname` 存在且为空目录，那么删除，然后重命名为 `newname`。注意 `newname` 不能包含 `oldname` 的前缀。
3. 若 `oldname` 或者 `newname` 引用符号链接，则处理符号链接本身。

# 15 符号链接
硬链接指向的是文件的 i 节点，而符号链接是对一个文件的间接指针。
函数原型
```c
# include <unistd.h>
int symlink(const char *actualpath, const char *sympath);
int symlinkat(const char *actualpath, int fd, const char *sympath);
```
# 16 文件的时间
- st_atim: 文件数据的最后访问时间
- st_mtim: 文件数据的最后修改时间
- st_ctim: i 节点状态最后修改的时间

修改文件访问和修改时间
```c
# include <sys/stat.h>
int futimesns(int fd, const struct timespec times[2]);
int utimensat(int fd, const char *path, const struct timespec times[2], int flag);
```
`times` 数组参数包含第一个元素包含访问时间，第二元素包含修改时间。
1. 如果 `times` 参数为空指针，则访问时间和修改时间设置为当前时间；
2. 如果 `times` 参数指向两个 `timespec` 结构，任一元素的的 `tv_nsec` 字段为 `UTIME_NOW`，相应的时间戳设置为当前时间；`tv_nsec` 字段的值为 `UTIME_OMIT`，则相应的时间保持不变。

# 17 `mkdir`，`mkdirat` 和 `rmdir` 函数
函数原型 
```c
# include <sys/stat.h>
int mkdir(const char *pathname, mode_t mode);
int mkdirat(int fd, const char *pathname, mode_t mode);
```
对于普通目录，至少有设置执行的权限位。

```c
# include <unistd.h>
int rmdir(const char *pathname);
```
删除一个空目录，如果目录的链接计数为 `0`, 则释放目录占用的空间。

# 18 `chdir`, `fchdir` 和 `getcwd` 函数
每个进程都有当前的工作目录，是搜索所有相对路径的起点。使用 `chdir` 和 `fchdir` 函数可以修改当前的工作目录
```c
# include <unistd.h>
int chdir(const char *pathname);
int fchdir(int fd);
```
调用该函数只影响调用它的进程。