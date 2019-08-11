---
date: 2019-07-01
status: draft
tags: APUE
title: APUE(06)
---

# 1 口令文件
`/etc/passwd` 是系统的口令文件，所有字段包含在 `<pwd.h>` 中定义的 `passwd` 文件
- char *pw_name: 用户名
- char *pw_paswd: 加密口令
- uid_t pw_uid: 用户ID
- gid_t pw_gid: 用户组ID
- char *pw_gecos: 注释字段
- char *pw_dir: 初始目录
- char *pw_shell: 初始shell
- char *pw_class: 初始访问类
- time_t pw_change: 下次更改口令
- time_t pw_expire: 账户有效期时间
对于登录项，有一下几点需要注意：
- 用户名为 root 账户，其用户 ID 为 0；
- 加密口令字段为占位符，存放在另一个文件中；
- 如果加密口令字段为空，则该用户没有口令；
- shell 字段为 `/dev/null` 表明不允许该用户登录，除此之外还有 `/bin/false` 和 `/bin/true` 阻止用户登录；
- 用户ID和组ID 为 `65534` 和 `65534` 允许方位人人皆可读、写的文件。

# 2 阴影口令
加密口令是经过非对称加密得到的用户口令的副本。某些系统将加密口令存放到阴影口令(shadow passwd) 文件中，通常为 `/etc/shadow` 文件。

# 3 组文件
器字段包含在 `<grp.h>` 定义的 `group` 结构中
- char *gr_name: 组名
- char *gr_passwd: 加密口令
- int gr_gid: 数值组 ID
- char **gr_mem: 指向个用户名指针的数组。

# 4 其他数据文件
- /etc/passwd: 口令
- /etc/group: 组
- /etc/shadow: 阴影
- /etc/hosts： 主机
- /etc/networks：网络
- /etc/protocols：协议
- /etc/services：服务

# 5 登录账户记录
`utmp` 文件记录当前登录到系统的各个用户；`wtmp` 文件跟踪各个登录和注销事件，所有文件包含下面的二进制记录：
```c
struct utmp{
    char ut_line[8]; /*tty line: "ttyh0", "ttyh1", ...*/
    char ut_name[8]; /*login name*/
    long ut_time;  /* seconds since epoch*/
}
```
`login` 程序填写次类型的结构，然后将其写入到 `utmp` 文件中，同时写入 `wtmp` 文件中。注销的时候， `init` 进程将 `utmp` 文件中相应的记录擦除，并将一个新的记录填写到 `wtmp` 文件中。

# 6 系统标识
函数原型
```c
# include <sys/utsname.h>
int uname(struct utsname *name);
```
传递一个 `ustname` 结构地址，然后返回。
```c
struct utsname {
    char sysname[]; // name of the operation system
    char nodename[]; // name of this node
    char release[]; // current release of operatiing system
    char version[]; // current version of thie release
    char machine[]; /// name the hardware type
};
```

# 7 时间
自协调世界时（`Coordinated Universal Time, UTC`): 公元 1970 年 1 月 1 日 00:00:00 这一特定时间经过的秒数。