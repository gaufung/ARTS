---
date: 2019-06-26
status: draft
tags: APUE
title: APUE(05)
---

# 1 流与 `FILE` 对象
当使用标准 `I/O` 库打开或者创建一个文件的时候，使用一个流与文件关联。可以用于单字节或者多字节（"宽")字符集，流的定向决定了所读、写字符是单字节还是多字节。设置流的定向：
```c
#include <stdio.h>
#include <wchar.h>
int fwide(FILE *fp, int mode);
```
- `mode` 参数为负，指定流是字节定向；
- `mode` 参数为正，指定流是宽定向的；
- `mode` 参数为 0，不设置流的定向，返回该流的定向。

# 2 标准输入、标准输出和标准错误
在 `<stdio.h>` 中定义好 `stdin, stdout, stderr` 分别指向文件描述符 `STDIN_FILENO, STDOUT_FILENO, STEDERR_FILENO`

# 3 缓冲
标准的 `I/O` 提供下面 3 种类型的缓冲
1. 全缓冲，在填满标准 `I/O` 缓冲区后才进行实际 `I/O` 操作，在第一次执行 `I/O` 操作时候，需要调用 `malloc` 函数获取缓冲区。`flush` 将缓冲区中的内容写到磁盘上。
2. 行缓冲，在输入和输出中遇到换行符，标准 `I/O` 执行 `I/O` 操作。
3. 不带缓冲，标准 `I/O` 库不对字符进行行缓存存储。

`ISO C` 要求下列特征
- 当且仅当标准输入和标准输出并不指向交互设备时，它们才是全缓冲；
- 标准错误绝不会是全缓冲。

使用下面函数更改函数缓冲类型
```c
# include <stdio.h>
void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);
```
`setbuf` 函数打开或者关闭缓冲机制；使用 `setvbuf` 可以 `mode` 参数实现：
- `_IOFBF` 全缓冲
- `_IOLBF` 行缓冲
- `_IONBF` 不带缓冲

# 4 打开流
打开标准 `I/O` 流
```c
# include <stdio.h>
FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE* restrict fp);
FILE *fdopen(int fd, const char  *type);
```
`type` 参数指定该 `I/O` 流的读、写方式。
- `r、rb`: 为读而打开 (`O_RDONLY`)
- `w、wb`： 把文件截断为 `0` 长度，或者写而创建 （`O_WRONLY|O_CREAT|O_TRUNC`).
- `a、ab`: 追加 (`O_WRONLY|O_CREAT|O_APPEND`)
- `r+、r+b、rb+` 为读和写打开 （`O_RDWR`)
- `w+、w+b、wb+` 把文件截断为0， 或者为读和写打开（`O_RDWR|O_CREAT|O_TRUNC`)
- `a+, a+b, ab+` 在文件尾读和写打开或者创建（`O_RDWR | O_CREAT|O_APPEND`)

# 5 读写流
1. 每次一个字符的 `I/O`;
2. 每次一行的 `I/O`， 分别使用 `fgets` 和 `fputs` 函数;
3. 直接 `I/O`，使用 `fread` 和 `fwrite` 函数支持类型的 `I/O`;

输入函数原型
```c
# include <stdio.h>
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
```
`getchar` 等同于 `getc(stdin) `， 如果读错或者已经到达文件末尾，则上述函数返回负数，通过下面函数判断上述情况的哪一种：
```c
# include <stdio.h>
int ferror(FILE *fp);
int feof(FILE *fp);
void clearerr(FILE *fp);
```
从流中读取数据后，可以调用 `ungetc` 将字符压会流中：
```c
# include <stdio.h>
int ungetc(int c, FILE *fp);
```

输出函数原型
```c
# include <stdio.h>
int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
```

# 6 每次一行 `I/O`
函数原型
```c
# include <stdio.h>
char *fgets(char *restrict buf, int n,  FILE *restrict fp);
char *gets(char* buf);
```
使用 `fgets`，必须指定缓冲区的长度 n，此函数一直读到下一个换行符为止，但是不超过 `n` 个字符。

输出一行的功能
```c
# include <stdio.h>
int fputs(const char *restrict str, FILE *restrict fp);
int puts(const char *str);
```
`fputs` 将以 `null` 字节终止的字符串写到指定流，`puts` 将一个以 `null` 字节终止的字符串写出标准输出。

# 7 二进制 `I/O`
函数原型
```c
# incldue <stdio.h>
size_t fread(void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
```
(1) 读写一个二进制数组，浮点数组的 2-5 个元素到文件上
```c
float data[10];
if (fwrite(&data[2], sizeof(float), 4 , fp)  != 4 )
    err_sys("fwrite error");
```

(2) 读写一个结构
```c
struct {
    short count;
    long total;
    char name[NAMESIZE];
} item;
fwrite(&item, sizeof(item), 1, fp);
```

# 8 定位流
（1） `ftell` 和 `fseek`
```c
# include <stdio.h>
long ftell(FILE *fp);
int fseek(FILE *fp, long offset, int whence);
int rewind(FILE *fp);
```
`ftell` 返回当前文件位置指示，`fseek` 定位一个二进制文件，指定字节的 `offset`， `whence` 有三种不同的方式。

（2） `ftello` 和 `fseeko`
```c
# include <stdio.h>
off_t ftello(FILE *fp);
int fseeko(FILE *fp, off_t offset, int whence);
```

(3) `fgetpos` 和 `fsetpos` 
由 `ISO C` 定义，注意跨平台使用
```c
# include <stdio.h>
int fgetpos(FILE *restrict fp, fpos_t *restrict pos);
int fsetpos(FILE *fp, const rpos_t *pos);
```

# 9 格式化输出
总共有 5 个 `printf` 函数来处理
```c
# include <stdio.h>
int printf(const char *restrict format, ...);
int fprintf(FILE *restrict fp, const char *restrict format, ...);
int dprintf(int fd, const char *restrict format, ...);
int sprintf(char *restrict buf, const char *restrict format, ...);
int snprintf(char *restrict buf, size_t n, const char *restrict format, ...);
```
`format` 格式如下
```
$[flags][fidwidth][precision][lenmodifier]convertypes
```

格式化输入：
```c
# incldue <stdio.h>
int scanf(const char *restrict format, ...);
int fscanf(FILE *restrict fp, const char *restrict format, ...);
int sscanf(const char *restrict buf, const char *restrict format, ...);
```
# 10 内存流
```c
# include <stdio.h>
FILE *fmemopen(void *restrict buf, size_t size, const char *restrict type);
```
`buf` 指向缓冲区开始的位置；`size` 参数指定了缓冲区大小的字节数；`type` 控制使用流。


```c
# include <stdio.h>
FILE *open_memstream(char **bufp, size_t *sizep);

# include <wchar.h>
FILE *open_wmemstream(wchar_t **bufp, size_t *sizep);
```
`open_memstream` 函数创建的流面向字节；`open_wmemstream` 函数创建的流面向宽字节。