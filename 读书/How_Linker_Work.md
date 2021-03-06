---
date: 2018-12-10
status: public
tags: CSAPP
title: 链接器是如何工作
---
# 1 编译程序
对于如下简单的C程序：
```c
// main.c
int sum(int *a, int n);
int array[2] = {1, 2};
int main()
{
    int val = sum(array, 2);
    return val
}
// sum.c
int sum(int *a, int n)
{
    int i, s = 0;
    for (i =0; i < n; i++){
        s += a[i]
    }
    return s;
}
```
使用`gcc`来编译成可执行程序：
```shell
gcc -Og -o prog main.c sum.c
```
那么完整的中间流程如下:
- C预处理器，将源程序`main.c`翻译成ASCII码中间文件
```shell
cpp [options] main.c /tmp/main.i
```
- C编译器，将`main.i`翻译成ASCII码的汇编语言文件`main.s`
```shell
cc1 /tmp/main.i -Og [options] -o /tmp/main.s
```
- 汇编器，将`main.s`翻译成可重定位文件`main.o`
```shell
as [options] -o /tmp/main.o /tmp/main.s
```
- 链接器，将可重定位文件和必要的系统文件合并起来，行程可执行文件
```shell
ld -o prog [system object files] /tmp/main.o /tmp/sum.o
```
- 加载器，将文件`prog`的代码和数据复制到内存中
```
linux> ./prog
```
# 2 目标文件
静态链接器以一组目标文件为输入，生成一个完全链接、可以加载和运行执行的目标文件。链接器完成两个工作：
- 符号解析：将目标文件中的每个符号引用和每个符号定义(`函数、全局变量、静态变量等`)关联起来；
- 重定位：每个符号定义与内存位置关联起来。
一般目标文件分为三种：
1. 可重定位目标文件
2. 可执行目标文件
3. 共享目标文件
在Linux或者Unix系统中，使用`ELF`(Executable and Linkable Format)作为目标文件的格式，典型的格式如下：

![](./_image/2018-12-10-21-24-42.jpg)
- `.text`:已编译的机器代码；
- `.rodata`:只读数据；
- `.data`: 已经初始化的全局和静态C变量；
- `.bss`:  未初始化的全局和静态C变量；
- `.symtab`:存放程序中定义和引用的函数和全局变量信息的符号表；
- `.rel.text`: 可重定位的`.text`节位置的列表；
- `.rel.data`:引用和定义的全局变量可从重定位的信息。
- ...

# 3 符号和符号表
每个模块`m`都有一个符号表，包含了三种符号：
- 由模块`m`定义并且能够被其他模块引用的全局符号；
- 由其他模块定义能够被模块`m`引用的全局符号；
- 只有模块`m`定义和引用的局部符号；
本地非静态成员变量由栈管理，但是带有C static的本地过程变量不是有栈管理，而是在`.data`或者`.bss`中分配空间。
`.symtab`节包含了`ELF`文件中的符号表，包含了一个条目，每个条目格式如下
```C
typedef struct {
    int name;
    char type:4, binding 4;
    char reversed;
    short section;
    long value;
    long size;
} Elf64_Symbol;
```
- name: 字符串表中的偏移，以`\0`结尾；
- value: 符号的地址，可重定位目标文件，为该节的偏移；对于可执行目标文件，为绝对地址；
- size: 目标的大小；
- type: 符号类型，要么是数据，要么是函数；
- biding: 是本地的，还是全局；
- section: 所属的小节。

# 4 符号解析
对于相同模块的的局部符号的定义，编译器能够保证只有一个定义，但是对于全局符号的引用，编译器假定在其他模块中定义。但是链接器的所有输入模块都找不到该符号的定义，那么输出错误。
在所有输入模块中，会存在全局符号的多重定义，但是分为强弱符号（函数，初始化的全局变量为强符号；未初始化的全局变量为弱符号)。链接器的采取的规则如下：
- 不允许有多个同名的强符号；
- 一个强符号和多个弱符号，选择强符号；
- 多个弱符号，随机选择其中一个弱符号。

# 5 静态库
在程序开发中，都会使用其他已经完成的模块，比如系统调用`printf`,`scanf`等，或者数学函数`sin`,`sqrt`等。如果这些模块都是属于可重定位的目标文件，那么在编译的时候做好需要将他们添加进来，有两种解决方案：
1. 将相似的功能的函数放在同一个目标文件中，比如`libc.o`文件，那么编译执行命令`gcc main.c /usr/lib/libc.o`。该方法的缺点是通常`libc.o`文件非常大，每次都会拷贝一份副本到执行程序中，增加负担。
2. 将每一个函数生成相应的目标文件，比如`/usr/lib/printf.o`, `/usr/lib/cos.o`, 那么编译执行命令`gcc main.c /usr/lib/printf.o /usr/lib/cos.o`。该方法的缺陷是导致目标文件特别多，增加了编译命令参数的复杂度。

将具有相似功能的目标文件进行归档`archive`, 生成一个目标文件的集合，有一个头部信息描述包含的目标文件的详细信息，这个文件也叫做静态库，以`.a`作为后缀名。在编译的过程中，从静态库中只提取所需的目标文件，这样既减少了生成文件的大小，也减低了编译的复杂度。
```shell
# create static library
gcc -c addvec.c multvec.c
ar rcs libvector.a addvec.o multvec.o
# link with given static library
gcc -c main.c
gcc -static -o prog main.o ./libvector.a
```
# 6 重定位
完成符号解析后，代码中每一个符号的引用的定义都关联起来，链接器合并所有输入的模块，并且为每一个符号分配运行时地址。编译器生成每个目标文件的时候，对于每个符号都会生成重定位条目，代码的重定位条目放在`.rel.text`中，已经初始化的条目放在`.rel.data`中。
在`ELF`中，可重定位的条目的格式如下
```C
typedef struct {
    long offset;
    long type:32,
         symbol: 32;
    long addend;        
}Elf64_Rela;
```
- offset: 需要修改的符号的节便宜；
- symbol: 标识被修改指向的符号；
- type: 如何修改引用：
    - R_X86_64_PC32: 使用32位PC相对位置引用；
    - R_X86_64_32: 使用32位绝对地址引用。
- addend: 符号常数。
整个重定位的过程也非常简单：首先链接器已经为每个节和每个符号确定的运行时的地址，分别用`ADDR(s)`和`ADDR(r.symbol)`表示。
- 计算每一个引用的实际地址：`refaddr = ADDR(s)+ r.offset`
- 如果是PC相对位置：`*refptr = (unsigned)(ADDR(r.symbol) + r.addend - refaddr)`
- 如果是PC绝对位置：`*refptr=(unsigned)(ADDR(r.symbol) + r.addend`
# 7 执行文件
典型的可执行`ELF`文件格式如下：
![](./_image/2018-12-15-11-25-13.jpg)
- `ELF`头描述文件总体格式，还有程序的入口；
- `.init`包含程序出事化调用的函数；
- `.text`，`.rodata`, `.data`等小节与可重定位目标文件中小节相同；
- 不含有可重定位小节
使用命令可以将可执行程序复制到内存中并执行
```shell
linux > ./prog
```
程序执行时内存映像如下图所示：

![](./_image/2018-12-15-11-34-42.jpg)
代码段总是从`0x400000`开始，然后是数据段，运行时的堆`heap`在数据段之后，通过`malloc`调用向上增长。用户栈从最大的合法用户位置 $2^{48}-1$，向较小内存位置增长；而$2^{48}$位置往上是内核中的代码和数据保留位置。
# 8 动态链接
在之前引入静态库虽然解决了不少问题，但是仍然有一下两个弊端：
- 如果升级了某个库，那么就需要重新连接使用了这个库的所有应用程序；
- 对于库函数调用，每一个运行程序都有一份拷贝，对于系统内存资源都是巨大的浪费。
共享库可以解决上述的问题，在运行和加载的时候，共享库可以加载到内存中任意位置，并且和内存中的程序链接起来，该过程称为动态链接。共享库也叫共享链接，在Linux中以`.so`后缀，在windows系统中以`.dll`后缀。
共享库以两种方式进行共享
- 任何文件系统，一个库只有一个`.so`文件；
- 一个共享库`.so`文件中`.text`节的一个副本，被多个不同运行程序共享。
如何生成共享库呢？
```shell
# build share library
liunx > gcc -shared -fpic -o libvector.so addevc.c. mulvec.c
# using share library
linux> gcc -o prog main.c  ./libvector.so
```
这样就创建了可执行目标文件`prog`, 其中`prog`中含有`.interp`小节，该小节包含了动态链接库的位置。

# 9 小结
从可重定位目标文件到可执行目标文件的中间过程是链接器所做的工作，符号解析基本是全部内容，将符号的引用于符号的定义关联起来，增加重定位的步骤。但是为了高效利用内存和方便的编译执行，引入静态库和共享库两个概念。