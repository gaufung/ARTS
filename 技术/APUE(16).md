---
date: 2019-08-01
status: public
tags: APUE
title: APUE(16）
---

# 1 套接字描述符

套接字是通信端点的抽象，正如使用文件描述符访问对象。
```c
# include <sys/socket.h>

int socket (int domain, int type, int protocol);
```
- `domain` 确定了通信的特征，每个域由 `AF_` 开头
    - `AF_INET`:  ipv4 
    - `AF_INET6`:  ipv6
    - `AF_UNIX`:  `UNIX` 域
    - `AF_UPSPEC`: 未指定

- `type` 指定套接字的类型
    - `SOCK_DGRAM` : 固定长度，无连接，不可靠的报文传递
    - `SOCK_RAW`： IP  协议的数据接口报文
    - `SOCK_SEQPACKET`：固定长度的、有序的、可靠的、面向连接的报文传递
    - `SOCK_STREAM`：有序的、可靠的、双向的、面向连接的字节流

# 2 寻址
每个地址标识都会被与特定的域相关，转换为如下结构
```c
struct scoketaddr {
  sa_familly_t sa_family;
  char sa_data[];  
};
```

地址查询一般在 `/etc/hosts` 或者 `/etc/services` 中，也可以通过 `DNS` 系统获得。每个服务都是有一个众所周知的端口号支持，可以从服务到端口号，也可以从端口号到服务名。可以使用 `bind` 函数将一个服务和套接字关联在一起。
```c
# include <sys/socket.h>
int bind(int sockfd, const struct *addr, socklen_t len);
```
- 进程正在运行的计算机上；
- 地址必须和创建套接字的地址族所支持的格式的相匹配；
- 地址中的端口必须不小于 1024；
- 一般只能将一个套接字绑定到一个端口上。

# 4 建立连接
如果是面向连接的网络服务，需要在交换数据之前建立连接
```c
# include <sys/socket.h>
int connect(int sockfd, const struct socketaddr *addr, socketlen_t len);
```
一但建立连接，可以调用 `listen` 函数表明愿意接受请求
```c
# include <sys/socket.h>
int listen(int sockfd, int backlog);
```
一旦调用了 `listen`, 所用的的套接字就能接受连接请求：
```c
# include <sys/socket.h>
int accept(int sockfd, struct socketaddr *restrict addr, socklen_t *restrict len);
```
# 5 数据
**发送数据**
```c
# include <sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t nbytes, int flags);
```

`send` 成功返回，表明数据已经无错误的发送到网络驱动程序上。

**接受数据**

```c
# include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t nbytes, int flags);
```

# 6 带外数据
带外数据（out-of-band data) 是一种通信协议所支持的可选功能。TCP 只支持一个字节的带外数据，在发送的时候指定 `MSG_OOB` 。


# 7 UNIX 域套接字
`Unix` 域套接字用于在同一台计算机上运行的进程之间的通信，因为仅仅是复制数据，因此效率更高。
```c
# incldue <sys/socket.h>

int socketpair(int domain, int type, int protocal, int sockfd[2]);
```

![](./_image/2019-08-01-20-17-46.jpg?r=50)

命名 `Unix` 域套接字
```c
struct sockaddr_run{
    sa_family sun_family; /* AF_UNIX */
    char sun_path[108];  /* pathname */
}
```
该成员包含一个路径名，该文件仅仅是像客户程序告示该套接字名字，无法打开也无法有应用程序进行通信。


