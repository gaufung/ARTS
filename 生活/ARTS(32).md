---
date: 2019-05-27
status: public
tags: ARTS
title: ARTS(32)
---

# 1 Algorithm
> 给定一个数组，有两个元素只出现了一次，而其余的出现了两次，找出两个只出现一次的元素，要求时间复杂度为 $O(n)$，空间复杂度为 $O(1)$。

对于位运算异或(`^`)，有如下性质

- `a ^ a = 0`
- `0 ^ a = a`
- `a ^ b ^ c = a ^ (b ^ c)`

如果数组中的元素都出现两次，那么对其所有元素进行异或操作最终结果必然为 `0`，如果元素中只有一个元素只出现了一次，那么所有元素异或的结果就是只出现一次的元素。那么两个元素只出现一次，那么该如何解决呢？
通过上述性质我们可以得知，最终的异或结果肯定不等于 `0`，也就是说肯定至少一位 `bit` 为 `1`，而且只出现一次的两个元素在该比特位数值不相同。所以通过这一个比特位将数组元素划分为两组，只出现一次的元素元素分别划分到这个两组中。按照上一步解决问题的方法，就可以找出只出现一次的元素。

```go
func onlyOnce(nums []int) []int{
    res := 0
    for _, num := range nums {
        res ^= num
    }
    i:=0
    for {
       if res & (mask <<i)  == (mask << i){
           break
       }
       i++
    }
    res1, res2 := 0, 0
    for _, num := range nums {
       if num & (mask << i) == (mask <<i) {
           res1 ^= num
       }else{
           res2 ^= num
       }
    }
    return []int{res1, res2}
}
```

# 2 Review

[Life of Binary](https://kishuagarwal.github.io/life-of-a-binary.html)

**二进制文件的前世今生**

## 2.1 概览
几乎你们中所有人都编写过程序，编码、编译和运行一气呵成，接下来就可以享受代码运行的成果。但是除此之外，你还需要感谢你的编译器在背后默默所做的工作（当然前提假设你在使用编译语言，而不是解释型语言）。
在本篇文章中，我将尝试向你展示你编写源代码如何转换成你的机器能够运行的文件。我选择 Linux 作为我的宿主机和 C 语言作为代码示例，这里涉及到的概念可以很方便的推广至其他编译型语言。
**注意：** 如果你想一步步尝试这篇文章的步骤，你需要确保在你本地机器上已经安装好 `gcc` 和 `elfutils` 两个工具。
接下来以一个简单的 C 语言程序为例，观察它是如何被编译器一步步转换的。

```c:n
#include <stdio.h>

// Main function
int main(void){
    int a = 1;
    int b = 2;
    int c = a + b;
    printf("%d\n", c);
    return 0;
}
```

这个程序创建两个变量，将它们加起来然后在屏幕上将结果打印出来。是不是很简单？但是我们可以通过这个简单的程序来了解到程序最终是如何在机器上执行的。
编译器的工作一般包含以下五个步骤：
![](./_image/2019-05-27-17-29-00.jpg)
接下来让我们一步步了解这些步骤并且发现更多的细节。

## 2.2 预处理
![](./_image/2019-05-28-09-01-48.jpg)
预处理阶段是由预处理器完成的，预处理器所做的工作就是来处理所有代码中所有预处理标识。这些标识符是以 `#` 开头，但是在处理它们之前，它首先移除代码中所有的注释，它们仅仅是给人来阅读的。然后它寻找所有的 `#` 命令，并且执行这些命令。
在上述代码中，我们只使用了 `#include` 指令，它就是告诉预处理器将 `stdio.h` 文件的拷贝至当前文件所在的位置。
你可以通过给 `gcc` 传递 `-E` 标识就可以看到预处理器的输出。
```shell
gcc -E sample.c
```
输出如下
```c:n
# 1 "sample.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "sample.c"

# 1 "/usr/include/stdio.h" 1 3 4

-----omitted-----


# 5 "sample.c"
int main(void) {
    int a = 1;
    int b = 2;
    int c = a + b;
    printf("%d\n", c);
    return 0;
}
```

## 2.3 编译
![](./_image/2019-05-28-09-02-17.jpg)
令人不解的是第二步也叫做编译，编译器从预处理的输出进行以下重要的工作。
- 将输出内容传递给词法分析器来生成不同的 `token`。`token` 是用来表示程序中诸如 `int`, `return`, `void`, `0` 等等。词法分析器也将每一个 `token` 同它所属的类型关联起来，无论它是字面字符串、整数或者浮点数。
- 将词法分析器的输出传递给语法分析器来检查编写的程序是否满足编程语言的语法规则，比如下面的一行代码将会引发语法错误
```c
b = a + ;
```
因为这里的 `+` 缺失一个操作数。
- 将语法分析器的输出传递给语义分析器，来检查程序是否满足类型检查、使用前需声明等规则。
- 如果程序语法正确，那么程序源码将会转换为目标机器架构的汇编指令。默认它会以当前运行机器为目标机器。当然如果你要为嵌入式系统构建程序，你可以将目标机器的架构传递给构建命令，`gcc` 可以为该机器生成汇编指令。

为了得到这一步的输出，传递 `-S` 标识至 `gcc` 编译器。
```
gcc -S sample.c
```
输出结果如下，这个取决于你的环境
```asm:n
    .file   "sample.c"                              // name of the source file
    .section    .rodata                             // Read only data
.LC0:                                               // Local constant
    .string "%d\n"                                  // string constant we used
    .text                                           // beginning of the code segment
    .globl  main                                    // declare main symbol to be global
    .type   main, @function                         // main is a function
main:                                               // beginning of main function
.LFB0:                                              // Local function beginning
    .cfi_startproc                                  // ignore them
    pushq   %rbp                                    // save the caller's frame pointer
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp                              // set the current stack pointer as the frame base pointer
    .cfi_def_cfa_register 6
    subq    $16, %rsp                               // set up the space
    movl    $1, -12(%rbp)
    movl    $2, -8(%rbp)
    movl    -12(%rbp), %edx
    movl    -8(%rbp), %eax
    addl    %edx, %eax
    movl    %eax, -4(%rbp)
    movl    -4(%rbp), %eax
    movl    %eax, %esi
    movl    $.LC0, %edi
    movl    $0, %eax
    call    printf
    movl    $0, %eax
    leave
    .cfi_def_cfa 7, 8
    ret                                             // return from the function
    .cfi_endproc
.LFE0:
    .size   main, .-main                            // size of the main function
    .ident  "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.4) 5.4.0 20160609"
    .section    .note.GNU-stack,"",@progbits        // make stack non-executable
```
如果你之前没有接触过汇编语言，这个看上去很吓人，但是也没有那么恐怖。相比较于常见的高级语言，理解汇编代码需要花更多的时间，如果时间充裕，我想你肯定能够读懂它。
让我们来看看这个文件包含了什么。
所有以 `.` 开头行都是汇编命令。`.file` 说明源文件名称，用来进行辅助 debug 。源码中的 `%d\n` 字面字符串被保存在 `.rodata` 节中（`ro` 意味着只读），因为它只读字符串。编译器将这个字符串命名为 `LC0`，在后面的代码可以引用这个。无论什么时候遇到以 `.L` 开头的标记，它都表明这些标识都是当前文件的本地，在其他文件中是不可见的。
`.global` 告诉我们 `main` 是全局符号，这也就意味着 `main` 函数可以被其他文件中调用。`.type` 告诉我们 `main` 是一个函数。接下来 `main` 函数的汇编代码。你可以忽视所有以 `cfi` 开头的指令，它们是用来在异常发生的时候进行栈展开时候用的。在本篇文章中我将会忽略这些，但是如果你想了解更多，可以参考如下[链接](https://sourceware.org/binutils/docs/as/CFI-directives.html)。
现在让我们详细探讨 `main` 函数汇编代码。
![](./_image/2019-05-27-19-10-54.jpg)
`11.` 你应当知道当你在调用函数的时候，必须要为这个函数调用创建一个栈帧。为了保证栈帧，在调用函数返回时候，我们需要知道该函数栈帧所指的位置。这也就是为什么我么需要将当前栈帧指针，也就是保存在 `rbp` 寄存器中的值压入栈中。
`14` 将当前栈指针移动到基指针，它变成了当前函数的栈指针。上图（1）示意当将 `rbp` 寄存器入栈之前的转态；图（2）展示了之前栈帧入栈后栈帧指针移动到当前函数栈帧指针的情况。
`16` 我们程序中包含了 3 个局部变量，它们类型都是整型。在我的机器上，每个 int 型占用 4 个字节，所以我们需要在栈上分配 12 个字节来保存局部变量。为局部变量在栈上分配空间只需要将栈指针减少到需要的字节数，原因是栈增长的方向是从高地址向低地址生长。但是你可以看到我们在这里减少了 16 而不是 12，这里的原因是空间分配的单位就是 16 字节。所以即使你需要一个局部变量，也有 16 字节空间在栈上分配，这样做的原因是在一些架构上有更好的性能表现。图（3） 展示了现在栈空间分布情况。
`17-22` 这部分代码非常直接，编译器使用 `rbp-12` 槽位作为变量 a 存储空间，`rbp-8` 作为 b 存储空间， 同样地使用 `rbp-4` 为 c 保留空间。然后分别将值 1 和值 2 放到 a 和 b 的地址。为了准备加法，将 b 值移动到 `edx` 寄存器，然后将 a 值移动 `eax` 寄存器。加法的结果值保存到 `eax` 寄存器，最后将结果保存到变量 c 的地址。
`23-27` 然后我们准备 `printf` 调用，首先将 c 变量的值移动到 `esi` 寄存器中，然后将字符串常量 `%d\n` 的地址移动到 `edi` 寄存器。现在 `esi` 和 `edi` 寄存器保存 `printf` 函数调用的参数， `edi` 保存到第一个参数，`esi` 保存第二个参数。然后我们调动 `printf` 函数来将变量 c 格式化成整型。特别要注意的是在这里 `printf` 符号还没有定义，接下来我们会看到如何解析 `printf` 符号。
`.size` 指出 `main` 函数字节大小；`.-main` 是一个表达式，而 `.` 符号表示当前行的地址。所以当前的表示是计算 `当前行的地址` - `main 函数地址`，结果就是 `main` 函数的字节大小。
`.dent` 告诉汇编器将下面行添加到 `.comment` 节。`.note.GNU-stack` 用来表明当前栈是否可执行，大部分情况下这是空字符串，这表明这个栈是不能执行的。

## 2.4 汇编
![](./_image/2019-05-28-09-01-19.jpg)
现在我们的程序已经是汇编语言了，但是处理器还是不能理解它们，我们需要将汇编代码转换成机器代码，而这些工作是由汇编器完成的。汇编器输入你的汇编文件然后生成目标文件，目标文件包含了你程序中的机器指令。
现在我们开始进行这一步操作，为了得到目标文件，需要将 `c` 标记传递个 `gcc` 编译器。
```shell
gcc -c sample.c
```
你将会得到的以 `o` 为扩展名的目标文件，这一次无法用普通的文本编辑器来打开它们，但是我们可以借助工具来查看究竟里面包含哪些内容。
目标文件有很多不同的文件格式，在这里我们专注特定的一种，也就是 Linux 平台使用的 [`ELF`](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 文件格式。
`ELF` 文件包含以下信息：
-  `ELF` 文件头
- 程序头表
- 节头表
- 引用前面表的其他数据

`ELF` 文件头包含目标文件的元信息，比如文件类型、编译机器类型、版本号、文件头大小等等。为了查看文件头，我们需要为 `eu-readelf` 工具传递 `-h` 标识。
```shell:n
$ eu-readelf -h sample.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Ident Version:                     1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           AMD x86-64
  Version:                           1 (current)
  Entry point address:               0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          704 (bytes into file)
  Flags:                             
  Size of this header:               64 (bytes)
  Size of program header entries:    0 (bytes)
  Number of program headers entries: 0
  Size of section header entries:    64 (bytes)
  Number of section headers entries: 13
  Section header string table index: 10
```
从上面列出来的，我们可以知道我们文件不包含任何程序头表，是符合预期的，因为程序头表只存在可执行文件和共享文件中。在下面的链接步骤，我们就可以看到程序头表。
在这里我们包含 13 个节，为了查看这些节，我们传递 `-S` 标识。
```shell:n
$ eu-readelf -S sample.o
There are 13 section headers, starting at offset 0x2c0:

Section Headers:
[Nr] Name                 Type         Addr             Off      Size     ES Flags Lk Inf Al
[ 0]                      NULL         0000000000000000 00000000 00000000  0        0   0  0
[ 1] .text                PROGBITS     0000000000000000 00000040 0000003c  0 AX     0   0  1
[ 2] .rela.text           RELA         0000000000000000 00000210 00000030 24 I     11   1  8
[ 3] .data                PROGBITS     0000000000000000 0000007c 00000000  0 WA     0   0  1
[ 4] .bss                 NOBITS       0000000000000000 0000007c 00000000  0 WA     0   0  1
[ 5] .rodata              PROGBITS     0000000000000000 0000007c 00000004  0 A      0   0  1
[ 6] .comment             PROGBITS     0000000000000000 00000080 00000035  1 MS     0   0  1
[ 7] .note.GNU-stack      PROGBITS     0000000000000000 000000b5 00000000  0        0   0  1
[ 8] .eh_frame            PROGBITS     0000000000000000 000000b8 00000038  0 A      0   0  8
[ 9] .rela.eh_frame       RELA         0000000000000000 00000240 00000018 24 I     11   8  8
[10] .shstrtab            STRTAB       0000000000000000 00000258 00000061  0        0   0  1
[11] .symtab              SYMTAB       0000000000000000 000000f0 00000108 24       12   9  8
[12] .strtab              STRTAB       0000000000000000 000001f8 00000016  0        0   0  1
```
你不必知道上述所有的节内容，每一个节都包含了很多信息，比如名称、大小和偏移量。下面是些重要节的介绍：
- `text` 节包含了机器代码；
- `rodata` 节包含了我们程序中包含只读数据，比如程序中常量和字面字符串，在这里它只包含 `%d\n` 字符串；
- `data` 节包含程序中已经初始化的数据，在这例子中是空是因为程序中不包含已初始化的数据；
- `bss` 节和 `data` 节差不多，但是它包含程序中未初始化的数据。未初始化的数据比如 `int arr[100]` 的声明，它就包含在 `bss` 节中。有一点需要注意的是 `bss` 节和其他节不同，它只包含该节占用大小而没有其他内容任何东西。这样做的原因是在加载的时候，这些所需要分配空间的数据都会在这一节分配，这样就能减小执行文件的大小；
- `strtab` 节包含程序所需要的全部字符串；
- `symtab` 节属于符号表，它包含程序中所有的符号；
- `rela.text` 节是重定位节，接下来我们会看到它。

你可以传递节的编号至 `en-readelf` 工具就可以查看该节的内容，同样也可以是使用 `objdump` 工具，它也能完成同样的工作。
让我们讨论一下 `rela.text` 节详细内容，还记得我们程序中使用的 `printf` 函数，我们知道这个函数并不是我们自己定义的，它是 C 语言库的一部分。正常情况下，当你编译的你的 C 程序的时候，编译器并不会将你调用的 C  函数包含进来，用来减少生成文件的大小，而是用一张表保存这些符号记录，叫做**重定位表** ，它们会被接下来的装载器填满。装载器我们接下来会讨论，现在首要任务是如果你查看 `rela.text` 节的时候，你会发现 `printf `函数，让我们再一次确认一下：
```shell:n
$ eu-readelf -r sample.o

Relocation section [ 2] '.rela.text' for section [ 1] '.text' at offset 0x210 contains 2 entries:
  Offset              Type            Value               Addend Name
  0x0000000000000027  X86_64_32       000000000000000000      +0 .rodata
  0x0000000000000031  X86_64_PC32     000000000000000000      -4 printf

Relocation section [ 9] '.rela.eh_frame' for section [ 8] '.eh_frame' at offset 0x240 contains 1 entry:
  Offset              Type            Value               Addend Name
  0x0000000000000020  X86_64_PC32     000000000000000000      +0 .text
```
你可以跳过第二个重定位节`.en_frame`，它是用来处理异常的，目前我们还不关心这一块。我们可以知道第一个重定位小节包含两个条目，一个是我们的 `printf` 符号，它告诉我们在我们的文件中有一个 `printf` 符号，当时它还没有被定义，这个符号在 `.text` 节偏移量为 `31`。让我们检查一下 `.text` 节中的偏移量为 `31` 字节的位置。
```asm:n
$ eu-objdump -d -j .text sample.o
sample.o: elf64-elf_x86_64

Disassembly of section .text:

       0:    55                       push    %rbp
       1:    48 89 e5                 mov     %rsp,%rbp
       4:    48 83 ec 10              sub     $0x10,%rsp
       8:    c7 45 f4 01 00 00 00     movl    $0x1,-0xc(%rbp)
       f:    c7 45 f8 02 00 00 00     movl    $0x2,-0x8(%rbp)
      16:    8b 55 f4                 mov     -0xc(%rbp),%edx
      19:    8b 45 f8                 mov     -0x8(%rbp),%eax
      1c:    01 d0                    add     %edx,%eax
      1e:    89 45 fc                 mov     %eax,-0x4(%rbp)
      21:    8b 45 fc                 mov     -0x4(%rbp),%eax
      24:    89 c6                    mov     %eax,%esi
      26:    bf 00 00 00 00           mov     $0x0,%edi
      2b:    b8 00 00 00 00           mov     $0x0,%eax
      30:    e8 00 00 00 00           callq   0x35               <<<<<< offset 0x31
      35:    b8 00 00 00 00           mov     $0x0,%eax
      3a:    c9                       leaveq
      3b:    c3                       retq
```
在这里我们可以看到在偏移量 `30` 的位置是操作码 `e8`，它是机器指令 `callq`，紧接着是   4 个字节，偏移量是 `0x31` 到 `0x34`，它应当是函数的 `printf` 的实际地址，但目前我们还没有这个函数，所以它们全部设置为 0。（后面我们会知道这个位置并不是 `printf` 函数的真实地址，而是通过 `plt` 进行间接调用，这个我们接下来会继续讨论）。

## 2.5 链接
![](./_image/2019-05-28-15-12-44.jpg)
到目前为止，我们所做的工作都是在单个文件上的，这个是不常见的。在实际生产中，你可能有成百上千的源代码文件，它们都需要被编译然后生成可执行文件。那么和我们现在例子有什么区别呢？
实际上所有的步骤都是一样的，所有的源代码文件都会被各自预处理、编译、汇编然后最终得到各自的目标文件。
既然每一个源代码文件都不是独立编写的，它们肯定有一些函数、全局变量是定义在其他文件中或者被其他文件所使用。
链接器的工作就是将所有的目标文件收集起来，遍历它们并且记录每一个文件中定义的符号和使用的符号。它可以从每一个目标文件中的符号表中收集这些信息，在收集到这些信息后，链接器创建一个文件，它是将每一个独立的目标文件的相应的节合并起来，这样所有的重定位的符号都可以被解析。
在我们的例子中，我们没有很多源文件，仅仅只有一份文件。但是因为我们使用了 C 语言库中的 `printf` 函数，我们的源文件会动态链接 C 语言库。现在我们链接我们的程序进一步研究它的输出。
```shell
gcc sample.c
```
我不会探讨更多的细节，因为它同样也是 `ELF` 文件只不过增加新的节。值得一点需要注意的是，当我们从汇编器中得到的目标的文件地址是相对地址；但是当我们链接了全部文件，我们得到文件是绝对地址。
在这个步骤中，链接器确定了我们程序中使用的全部符号，谁使用了这些符号，谁定义了这些符号。链接器将符号定义的地址位置映射到符号使用的位置。经历这一步，仍然还会有一些符号还没有被解析，比如我们使用的 `printf` 符号。通常有一些符号既不是外部定义的变量或者外部定义的函数。链接器会为这些符号创建重定位表，这个和之前汇编器创建的一样，只不过这个时候这些条目仍然没有被解析。
此时你需要知道一件事，你从其他库中使用的函数和数据可以被静态或者动态链接。静态连接意味着这些库中的函数和数据被拷贝粘贴到你可以执行文件中，而在动态链接中这些函数和数据并没有被拷贝到可执行文件中，以便减少最终执行文件大小。
对于库而言，为了能完成动态功能，它必须是一个共享库文件（so 文件）。正常情况下，被很多程序使用的公共库只作为共享库，比如 `libc` 库文件被很多程序使用，如果每个应用程序使用静态连接，在某一时刻，同样的代码会有很多副本在运行，这样占据内存空间。动态链接就是解决这样的问题，在任何时刻只有一份 `libc` 文件拷贝在运行，每一个引用程序都会应用这个共享库地址。
为了让动态链接工作，链接器创建两个在目标文件中不存在的节。它们是 `.plt` (Procedure Linkage Table)和 `.got`(Global Offset Table)节。我们会在装载的时候讨论着两个节，它们在装载可执行文件的时候非常有用。

## 2.6 装载

![](./_image/2019-05-28-16-11-34.jpg)

现在我们可以运行我们的可执行文件。
当你在 GUI 平台上双击可执行文件或者在命令行中运行它，间接调用了 `execev` 系统方法。它是系统调用，此时内核开始工作加载你的可以执行文件至内存。
还记得之前的程序头表，在这里非常有用

```asm:n
$ eu-readelf  -l a.out 
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000400040 0x0000000000400040 0x0001f8 0x0001f8 R E 0x8
  INTERP         0x000238 0x0000000000400238 0x0000000000400238 0x00001c 0x00001c R   0x1
	[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x000724 0x000724 R E 0x200000
  LOAD           0x000e10 0x0000000000600e10 0x0000000000600e10 0x000228 0x000230 RW  0x200000
  DYNAMIC        0x000e28 0x0000000000600e28 0x0000000000600e28 0x0001d0 0x0001d0 RW  0x8
  NOTE           0x000254 0x0000000000400254 0x0000000000400254 0x000044 0x000044 R   0x4
  GNU_EH_FRAME   0x0005f8 0x00000000004005f8 0x00000000004005f8 0x000034 0x000034 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x000e10 0x0000000000600e10 0x0000000000600e10 0x0001f0 0x0001f0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01      [RO: .interp]
   02      [RO: .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame]
   03      [RELRO: .init_array .fini_array .jcr .dynamic .got] .got.plt .data .bss
   04      [RELRO: .dynamic]
   05      [RO: .note.ABI-tag .note.gnu.build-id]
   06      [RO: .eh_frame_hdr]
   07     
   08      [RELRO: .init_array .fini_array .jcr .dynamic .got]
```

内核是如何在文件中找到这个表呢？当然是从 `ELF` 头中获取，因为它在文件中的偏移总是零。完成这一步，内核查看所有类型是 `LOAD` 的条目。然后将它们装载到进程内存空间。
正如你看到上面所列的，这里有两个条目的类型是 `LOAD`，你也可以发现这两个节被包含在同一个 `segment` 中。
现代操作系统和处理器以页的形式管理内存，计算机内存被划分成若干个大小相同的块。当应用程序需要内存，然后操作系统会分配若干个页给应用程序。这样做除了高效管理内存之外，还可以提供更好的安全性。操作系统和内核可以为每一个页分配保护位。保护位指定了该页是否只读、可以写或者可以执行。如果一个页的保护位设置为只读，那么就不能被修改，这样做的话可以防止有意或者无意的修改数据。
只读页可以运行多个运行同样程序的进程共用同样的页，因为这些页都是只读的，没有任何运行的程序可以修改这个页，所以每个页都可以正常运行工作。
为了实现这个保护位机制，我们需要告诉内核哪些页可以标记为只读、可写和执行，这些信息都被保存在上面每个条目的标记中。
注意第一个 `LOAD` 条目，它被标记为 `R` 和 `E`，这也就意味着该 `segment` 可以读和执行但是不能修改。如果你查看这个 `segment` 来自哪些节，你可以发现它们来自 `.text` 和 `.rodata`。也就说明我们的代码和只读数据仅仅只能读和执行，但是不能修改操作。
同样的，第二个 `LOAD` 条目包含了初始化和未初始化的数据，`GOT` 表被标记为 `RW`，所以它们可以被读写但是被被执行。
在加载完各个 `segment` 并设置权限后，内核检查是否有 `.interp` segment。在静态链接可执行文件中，这个 `segement` 是不需要的，因为可执行文件包含了它所需要的全部代码，但是对于动态链接可执行文件，这个 `segment` 是非常重要的。这个 `segement` 包含了动态链接器的路径。（你可以通过传递 `-static` 标识给 `gcc` 编译器来生成静态链接可执行文件，然后检查文件头表查看是否有 `.interp` segment)
在我们的例子中，你可以发现它指向的动态链接器的路径是 `/lib64/ld-linux-x86-64.so.2`。和我们的执行文件一样，内核开始通过读取头表来装载这些共享文件，找到它的 `segments` 然后在我们当前进程内存空间加载这个共享库。在静态连接可执行文件中，这一步是不需要的，内核直接将控制权交给我们的程序。但是在这里内核将控制权交给动态链接器然后将 `main` 函数的地址压入到被调用栈中，所以当动态链接器完成它的工作，它知道如何将控制权交出去。
你应当知道我们之前跳过的两个表 `Procedure Linkage Table` 和 `Global Offset Table`，因为它们和动态链接器功能关联非常大。
程序中有两种类型的重定位：变量重定位和函数重定位。对于外部定义的变量，我们在 `GOT` 表中包含这个条目，同样的对于外部定义的函数也包含在这张表中。总之，`GOT`包含了全部外部定义的变量和函数，而 `PLT1` 表仅仅是外部函数条目。接下来的例子可以解释为什么函数拥有两个条目。
我们以 `printf` 函数为例，看看这些表是如何工作的。在我们的 `main` 函数中，我们可以看看调用 `printf` 函数的指令。

```asm
400556:    e8 a5 fe ff ff           callq   0x400400
```

调用指令的调用的地址是 `.plt` 节中的一部分，让我们详细看看

```asm:n
$ objdump -d -j .plt a.out
a.out:     file format elf64-x86-64

Disassembly of section .plt:

00000000004003f0 <printf@plt-0x10>:
  4003f0:	ff 35 12 0c 20 00    	pushq  0x200c12(%rip)        # 601008 <_GLOBAL_OFFSET_TABLE_+0x8>
  4003f6:	ff 25 14 0c 20 00    	jmpq   *0x200c14(%rip)        # 601010 <_GLOBAL_OFFSET_TABLE_+0x10>
  4003fc:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000400400 <printf@plt>:
  400400:	ff 25 12 0c 20 00    	jmpq   *0x200c12(%rip)        # 601018 <_GLOBAL_OFFSET_TABLE_+0x18>
  400406:	68 00 00 00 00       	pushq  $0x0
  40040b:	e9 e0 ff ff ff       	jmpq   4003f0 <_init+0x28>

0000000000400410 <__libc_start_main@plt>:
  400410:	ff 25 0a 0c 20 00    	jmpq   *0x200c0a(%rip)        # 601020 <_GLOBAL_OFFSET_TABLE_+0x20>
  400416:	68 01 00 00 00       	pushq  $0x1
  40041b:	e9 d0 ff ff ff       	jmpq   4003f0 <_init+0x28>
```

对于每一个外部定义的函数，我们都会在 `plt` 节中保留相关的条目，除了第一个条目，它们看上去都很像。它是一个特殊的条目，接下来我们会讨论这个。
这里我们发现跳转到 `0x601018` 位置包含的地址，这个地址是 `GOT` 表的一个目录，让我们看看这个地址的内容。

```shell:n
$ objdump -s a.out | grep -A 3 '.got.plt' 
Contents of section .got.plt:
601000 280e6000 00000000 00000000 00000000  (.`.............
601010 00000000 00000000 06044000 00000000  ..........@.....
601020 16044000 00000000                    ..@..... 
```

在这里神奇的事情发生，除了第一次 `printf` 函数被调用，这个地址的值就是 C 语言库中 `printf` 的真实地址。我们只需要简单的跳转相应的位置即可，但是第一次，还有其他的事情要做。
当 `printf` 函数第一次被调用，`plt` 中 `printf` 函数的条目中下一个指令位置被保存在这个位置。从上面列出来的可以看出是 `400406`，是通过小段编码的形式保存。在 `plt` 条目的位置，我们需要将指令也就是 `0` 入栈。每一个 `plt` 条目都有相同的指令但是压入不同的数字。`0` 意味着 `printf` 函数在重定位表中的偏移。压入指令紧跟着的是跳转到第一个指令的 `plt` 条目。
从上面知道，我告诉你第一个条目非常特殊，因为它用来让动态链接器来解析外部符号和重定位它们。为了完成这个功能，我们跳转到 `got` 表中包含地址 `601010` 的地址。此时这些条目都被填充 `0`，但是这个地址会在运行的和内核调用动态链接器的时候修改。
当过程被调用，链接器会从外部共享对象中解析它先前压入的符号，将符号正确的地址写入 `got` 表中，所以当 `printf` 函数被调用后，我们不必要再次查询链接器，我们可以直接从 `plt` 中跳转到 `C` 语言库中的 `printf` 函数。
这个过程叫做延迟加载，一个程序可能包含很多外部符号，但是并不是在程序都会调用它们，所以符号解析会被推迟到真正调用的时候，这样可以节省我们程序启动的时间。
从上面的讨论中，我们从未修改 `plt` 节，只修改了 `got` 节。这也就是为什么第一个 `LOAD` segment 被标记为只读，而第二个包含 `got` 节的 `LOAD` segement 被标记为写。
这就是动态链接器的工作方式，我已经跳过了很多细节，如果你对这些细节非常感兴趣，可以查看这篇[文章](https://www.akkadia.org/drepper/dsohowto.pdf)。
让我们回过头查看程序加载，我们完成了大部分工作。内核已经加载了全部可加载的 `segment`, 已经启动动态链接器。剩下的工作就是唤醒 `main` 函数。在链接器工作结束，所有的准备工作已经完成了，当我们调用 `main` 函数，我们从终端得到如下的输出。
```shell
3
```
# 3 Tip

**进程管理工具**

## 3.1 查询进程
查询用户 `colin115` 的进程
- ps -ef | grep colin115
- ps -lu colin115

查看某一进程
- pgrep -l re

查看进程详细信息
- ps -ajx

## 3.2 losf

查看端口使用情况
lsof -i:3306
查看用户打开的文件
lsof -u username
查看进程 init  打开的文件
lsof -c <init>
查看进程 id 打开的文件
lsof -p <pid>

**性能监控**
- sar -u 1 2: 查看 CPU 使用率，每隔 1 秒查看一次，共 2 次；
- sar -q 1 2: 查看队列中的进程数， 每隔 1 秒查看一次，共 2 次；
- sar -r 1 2: 查看内存使用， 每隔 1 秒查看一次， 共 2 次；
- sar -W 1 2: 查看页交换，每一个1 秒查看一次，共 2 次；

**网络工具**

- netstat -a: 列出所有的端口号
- netstat -at: 列出所有的 tcp 端口号
- ping <ip>: ping IP 地址
- traceroute <ip>: 连接 <ip> 经历的路由
- host <domian>: 查看 IP 
- host <ip>: 查看域名

**ftp sftp ssh**

- ssh Id@host
- sftp Id@host



# 4 Share