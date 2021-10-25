# 第六章 运行时的检查与调试

调试器是一种可以检查正在执行的程序的程序。调试器可以启动并执行一个程序，然后在某个特定的行停止，方便我们检查此时此刻程序的状态。调试暂停（但没有结束）的位置被称为断点。

我们将使用`GDB`——GNU调试器来调试我们的内核。`gdb`是程序的名字，可以完成下面四个主要工作：

* 启动你的程序，并指明任何会影响其行为的东西。
* 使你的程序在指定的条件下暂停。
* 当程序终止时，可以检查到底发生了些什么。
* 改变程序的状态，你可以通过试验尝试纠正某个错误的影响，接着再了解下一个。

## 6.1 一个示例程序

必须要有一个现成的程序才能进行调试。对于这一章来说，我们的老朋友"Hello World"就够用了：

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello World!\n");
    return 0;
}
```

编译时，我们用选项`-g`把调试信息也保存下来。

We compile it with debugging information with the option -g:

```bash
$ gcc -m32 -g hello.c -o hello
```

最后，我们启动`gdb`，并且把程序作为启动参数：

```bash
$ gdb hello
```

## 6.2 对一个程序进行静态检查

在检查一个运行中的程序之前，`gdb`首先会加载它。加载到内存之后（即使不运行），已经可以获取到很多有用的信息以供检查。本节中的命令可以在程序运行前使用。然而，它们也可以在程序运行时使用，并且可以显示更多的信息。

### 6.2.1 命令：`info target`/`info file`/`info files`

下面的命令可以打印出正在调试的目标的信息。目标就是正在调试的程序。

示例6.2.1 `hello`程序的命令输出，详尽地展示了一个本地的目标。

```bash
(gdb) info target
```

```text
Symbols from "/tmp/hello".
Local exec file:
  `/tmp/hello', file type elf32-i386.
  Entry point: 0x8048310
  0x08048154 - 0x08048167 is .interp
  0x08048168 - 0x08048188 is .note.ABI-tag
  0x08048188 - 0x080481ac is .note.gnu.build-id
  0x080481ac - 0x080481cc is .gnu.hash
  0x080481cc - 0x0804821c is .dynsym
  0x0804821c - 0x08048266 is .dynstr
  0x08048266 - 0x08048270 is .gnu.version
  0x08048270 - 0x08048290 is .gnu.version_r
  0x08048290 - 0x08048298 is .rel.dyn
  0x08048298 - 0x080482a8 is .rel.plt
  0x080482a8 - 0x080482cb is .init
  0x080482d0 - 0x08048300 is .plt
  0x08048300 - 0x08048308 is .plt.got
  0x08048310 - 0x080484a2 is .text
  0x080484a4 - 0x080484b8 is .fini
  0x080484b8 - 0x080484cd is .rodata
  0x080484d0 - 0x080484fc is .eh_frame_hdr
  0x080484fc - 0x080485c8 is .eh_frame
  0x08049f08 - 0x08049f0c is .init_array
  0x08049f0c - 0x08049f10 is .fini_array
  0x08049f10 - 0x08049f14 is .jcr
  0x08049f14 - 0x08049ffc is .dynamic
  0x08049ffc - 0x0804a000 is .got
  0x0804a000 - 0x0804a014 is .got.plt
  0x0804a014 - 0x0804a01c is .data
  0x0804a01c - 0x0804a020 is .bss
```

输出显示了各种报告：

* 符号文件的路径。符号文件是包含着调试信息的文件。通常，它与二进制文件是同一个文件，但是更多情况是把可执行的二进制文件以及它的调试信息分成两个文件，尤其是在远程调试的时候。示例中，它是这一行：

```text
Symbols from "/tmp/hello".
```

* 调试程序的路径以及它的文件类型。示例中，它是这一行：

```text
Local exec file:
  `/tmp/hello', file type elf32-i386.
```

* 调试程序的入口。也就是程序运行的第一段代码。示例中，它是这一行：

```text
Entry point: 0x8048310
```

* 带有起始和结束地址的节表。示例中，它是报告的剩余部分。  

示例6.2.1 如果调试的程序运行在另外一台设备上，它就是一个远程目标，`gdb`只会打印一些简要的信息：

```bash
(gdb) info target
```

```text
Remote serial target in gdb-specific protocol:
Debugging a target over a serial line.
```

### 6.2.2 命令：`maint info sections`

这个命令与`info target`类似，但给出了关于程序各节的额外信息，特别是每个节的文件偏移量以及各个标志。

下面是针对`hello`程序运行时的输出。

```bash
(gdb) maint info sections
```

```text
Exec file:
    `/tmp/hello', file type elf64-x86-64.
  [0]     0x00400238->0x00400254 at 0x00000238: .interp ALLOC LOAD READONLY DATA HAS_CONTENTS
  [1]     0x00400254->0x00400274 at 0x00000254: .note.ABI-tag ALLOC LOAD READONLY DATA HAS_CONTENTS
  [2]     0x00400274->0x00400298 at 0x00000274: .note.gnu.build-id ALLOC LOAD READONLY DATA HAS_CONTENTS
  [3]     0x00400298->0x004002b4 at 0x00000298: .gnu.hash ALLOC LOAD READONLY DATA HAS_CONTENTS
  [4]     0x004002b8->0x00400318 at 0x000002b8: .dynsym ALLOC LOAD READONLY DATA HAS_CONTENTS
  [5]     0x00400318->0x00400355 at 0x00000318: .dynstr ALLOC LOAD READONLY DATA HAS_CONTENTS
  [6]     0x00400356->0x0040035e at 0x00000356: .gnu.version ALLOC LOAD READONLY DATA HAS_CONTENTS
  [7]     0x00400360->0x00400380 at 0x00000360: .gnu.version_r ALLOC LOAD READONLY DATA HAS_CONTENTS
....remaining output omitted....
```

这个输出与`info target`的输出类似，但是有更多的细节。在节的名字旁边是节的标志，它们是节的属性。在这里，我们可以看到带有`LOAD`标志的部分来自LOAD段。这个命令可以结合节的标志，对输出进行过滤：

*ALLOBJ* 显示所有加载的对象文件的节，包括共享库。共享库只有在程序已经运行时才会显示出来。

*section names* 只显示命名了的节。

示例6.2.4 命令：

```bash
(gdb) maint info sections .text .data .bss
```

只显示`.text`、`.data`和`.bss`节：

```text
Exec file:
    `/tmp/hello', file type elf64-x86-64.
  [13]     0x00400430->0x004005c2 at 0x00000430: .text ALLOC LOAD READONLY CODE HAS_CONTENTS
  [24]     0x00601028->0x00601038 at 0x00001028: .data ALLOC LOAD DATA HAS_CONTENTS
  [25]     0x00601038->0x00601040 at 0x00001038: .bss ALLOC
```

*section-flags* 只显示具有指定标志的节。注意，尽管这些标志是基于节属性定义的，他们都是`gdb`特有的。目前，`gdb`理解下列标志：

&nbsp;&nbsp;&nbsp;&nbsp;*ALLOC* 节会在程序加载时在进程中分配获得空间。除了包含调试信息的节之外，剩下所有的节都有这个标志。

&nbsp;&nbsp;&nbsp;&nbsp;*LOAD* 节会从文件中加载到子进程内存中。预初始化的代码和数据都有设置这个标志，而`.bss`节没有。

&nbsp;&nbsp;&nbsp;&nbsp;*RELOC* 节在加载之前需要重新定位。

&nbsp;&nbsp;&nbsp;&nbsp;*READONLY* 节不能被子进程修改。

&nbsp;&nbsp;&nbsp;&nbsp;*CODE* 节只包含可执行代码。

&nbsp;&nbsp;&nbsp;&nbsp;*DATA* 节只包含数据（没有可执行代码）。

&nbsp;&nbsp;&nbsp;&nbsp;*ROM* 节会驻留在ROM中。

&nbsp;&nbsp;&nbsp;&nbsp;*CONSTRUCTOR* 节包含用于构造函数/析构函数列表的数据。

&nbsp;&nbsp;&nbsp;&nbsp;*HAS_CONTENTS* 节不为空。

&nbsp;&nbsp;&nbsp;&nbsp;*NEVER_LOAD* 是给链接器的指令，不会输出该节。

&nbsp;&nbsp;&nbsp;&nbsp;*COFF_SHARED_LIBRARY* 会通知链接器此节包含`COFF`共享库信息。`COFF`是一种对象文件格式，与`ELF`类似。`ELF`是可执行二进制文件的文件格式，而`COFF`是对象文件的文件格式。

&nbsp;&nbsp;&nbsp;&nbsp;*IS_COMMON* 节包含通用符号。

示例6.2.5 用下面的命令，我们可以限制输出，只显示包含代码的节：

```bash
(gdb) maint info sections CODE
```

输出是：

```text
Exec file:
    `/tmp/hello', file type elf64-x86-64.
  [10]     0x004003c8->0x004003e2 at 0x000003c8: .init ALLOC LOAD READONLY CODE HAS_CONTENTS
  [11]     0x004003f0->0x00400420 at 0x000003f0: .plt ALLOC LOAD READONLY CODE HAS_CONTENTS
  [12]     0x00400420->0x00400428 at 0x00000420: .plt.got ALLOC LOAD READONLY CODE HAS_CONTENTS
  [13]     0x00400430->0x004005c2 at 0x00000430: .text ALLOC LOAD READONLY CODE HAS_CONTENTS
  [14]     0x004005c4->0x004005cd at 0x000005c4: .fini ALLOC LOAD READONLY CODE HAS_CONTENTS
```

### 6.2.3 命令：`info functions`

这个命令列出了所有的函数名称以及它们的加载地址。我们可以用正则表达式来过滤这些名称。

示例6.2.6 执行命令，我们可以得到如下的输出：

```bash
(gdb) info functions
```

```text
All defined functions:
File hello.c:
int main(int, char **);
Non-debugging symbols:
0x00000000004003c8  _init
0x0000000000400400  puts@plt
0x0000000000400410  __libc_start_main@plt
0x0000000000400430  _start
0x0000000000400460  deregister_tm_clones
0x00000000004004a0  register_tm_clones
0x00000000004004e0  __do_global_dtors_aux
0x0000000000400500  frame_dummy
0x0000000000400550  __libc_csu_init
0x00000000004005c0  __libc_csu_fini
0x00000000004005c4  _fini
```

### 6.2.3 命令：`info variables`

这个命令列出了所有的全局变量和静态变量的名称，它们也可以用正则表达式进行过滤。

示例6.2.7 如果我们在示例源程序中添加一个全局变量`int i`并重新编译，然后运行该命令，我们会得到以下输出：

```bash
(gdb) info variables
```
  
```text
All defined variables:
File hello.c:
int i;
Non-debugging symbols:
0x00000000004005d0  _IO_stdin_used
0x00000000004005e4  __GNU_EH_FRAME_HDR
0x0000000000400708  __FRAME_END__
0x0000000000600e10  __frame_dummy_init_array_entry
0x0000000000600e10  __init_array_start
0x0000000000600e18  __do_global_dtors_aux_fini_array_entry
0x0000000000600e18  __init_array_end
0x0000000000600e20  __JCR_END__
0x0000000000600e20  __JCR_LIST__
0x0000000000600e28  _DYNAMIC
0x0000000000601000  _GLOBAL_OFFSET_TABLE_
0x0000000000601028  __data_start
0x0000000000601028  data_start
0x0000000000601030  __dso_handle
0x000000000060103c  __bss_start
0x000000000060103c  _edata
0x000000000060103c  completed
0x0000000000601040  __TMC_END__
0x0000000000601040  _end
```

### 6.2.4 命令：`disassemble/disas`

这个命令显示可执行文件的汇编代码。

示例6.2.8 `gdb`可以显示一个函数的汇编代码：

```bash
(gdb) disassemble main
```  

```text
Dump of assembler code for function main:
    0x0804840b <+0>:  lea    ecx,[esp+0x4]
    0x0804840f <+4>:  and    esp,0xfffffff0
    0x08048412 <+7>:  push   DWORD PTR [ecx-0x4]
    0x08048415 <+10>: push   ebp
    0x08048416 <+11>: mov    ebp,esp
    0x08048418 <+13>: push   ecx
    0x08048419 <+14>: sub    esp,0x4
    0x0804841c <+17>: sub    esp,0xc
    0x0804841f <+20>: push   0x80484c0
    0x08048424 <+25>: call   0x80482e0 <puts@plt>
    0x08048429 <+30>: add    esp,0x10
    0x0804842c <+33>: mov    eax,0x0
    0x08048431 <+38>: mov    ecx,DWORD PTR [ebp-0x4]
    0x08048434 <+41>: leave  
    0x08048435 <+42>: lea    esp,[ecx-0x4]
    0x08048438 <+45>: ret    
End of assembler dump.
```

如果包含了源代码，这个命令会更有用：

```bash
(gdb) disassemble /s main
```

```text
Dump of assembler code for function main:
hello.c:
4 {
    0x0804840b <+0>:  lea     ecx,[esp+0x4]
    0x0804840f <+4>:  and     esp,0xfffffff0
    0x08048412 <+7>:  push    DWORD PTR [ecx-0x4]
    0x08048415 <+10>: push   ebp
    0x08048416 <+11>: mov    ebp,esp
    0x08048418 <+13>: push   ecx
    0x08048419 <+14>: sub    esp,0x4
5     printf("Hello World!\n");
    0x0804841c <+17>: sub    esp,0xc
    0x0804841f <+20>: push   0x80484c0
    0x08048424 <+25>: call   0x80482e0 <puts@plt>
    0x08048429 <+30>: add    esp,0x10
6     return 0;
    0x0804842c <+33>: mov    eax,0x0
7 }
    0x08048431 <+38>: mov    ecx,DWORD PTR [ebp-0x4]
    0x08048434 <+41>: leave  
    0x08048435 <+42>: lea    esp,[ecx-0x4]
    0x08048438 <+45>: ret    
End of assembler dump.
```

这时，高阶的源代码（绿色文本）作为汇编的一部分被包含了进来。每一行代码下方都有对应的汇编代码。

示例6.2.10 如果加上`/r`选项，十六进制的原始指令也会包含进来，就像`objdump`默认会显示汇编代码一样：

```bash
(gdb) disassemble /rs main
```
  
```text
Dump of assembler code for function main:
hello.c:
4 {
    0x0804840b <+0>:  8d 4c 24 04     lea    ecx,[esp+0x4]
    0x0804840f <+4>:  83 e4 f0        and    esp,0xfffffff0
    0x08048412 <+7>:  ff 71 fc        push   DWORD PTR [ecx-0x4]
    0x08048415 <+10>: 55              push   ebp
    0x08048416 <+11>: 89 e5           mov    ebp,esp
    0x08048418 <+13>: 51              push   ecx
    0x08048419 <+14>: 83 ec 04        sub    esp,0x4
5     printf("Hello World!\n");
    0x0804841c <+17>: 83 ec 0c        sub    esp,0xc
    0x0804841f <+20>: 68 c0 84 04 08  push   0x80484c0
    0x08048424 <+25>: e8 b7 fe ff ff  call   0x80482e0 <puts@plt>
    0x08048429 <+30>: 83 c4 10        add    esp,0x10
6     return 0;
    0x0804842c <+33>: b8 00 00 00 00  mov    eax,0x0
7 }
    0x08048431 <+38>: 8b 4d fc        mov    ecx,DWORD PTR [ebp-0x4]
    0x08048434 <+41>: c9              leave  
    0x08048435 <+42>: 8d 61 fc        lea    esp,[ecx-0x4]
    0x08048438 <+45>: c3              ret    
End of assembler dump.
```
  
示例6.2.11 也可以指明输出某一个文件内的某个函数：

```bash
(gdb) disassemble /sr 'hello.c'::main
```

```text
Dump of assembler code for function main:
hello.c:
4 {
    0x0804840b <+0>:  8d 4c 24 04     lea    ecx,[esp+0x4]
    0x0804840f <+4>:  83 e4 f0        and    esp,0xfffffff0
    0x08048412 <+7>:  ff 71 fc        push   DWORD PTR [ecx-0x4]
    0x08048415 <+10>: 55              push   ebp
    0x08048416 <+11>: 89 e5           mov    ebp,esp
    0x08048418 <+13>: 51              push   ecx
    0x08048419 <+14>: 83 ec 04        sub    esp,0x4
5     printf("Hello World!\n");
    0x0804841c <+17>: 83 ec 0c        sub    esp,0xc
    0x0804841f <+20>: 68 c0 84 04 08  push   0x80484c0
    0x08048424 <+25>: e8 b7 fe ff ff  call   0x80482e0 <puts@plt>
    0x08048429 <+30>: 83 c4 10        add    esp,0x10
6     return 0;
    0x0804842c <+33>: b8 00 00 00 00  mov    eax,0x0
7 }
    0x08048431 <+38>: 8b 4d fc        mov    ecx,DWORD PTR [ebp-0x4]
    0x08048434 <+41>: c9              leave  
    0x08048435 <+42>: 8d 61 fc        lea    esp,[ecx-0x4]
    0x08048438 <+45>: c3              ret    
End of assembler dump.
```

文件名必须包含在单引号中，且函数必须以双冒号为前缀，比如`'hello.c'::main`指定对文件`hello.c`中的函数`main`进行反汇编。

### 6.2.6 命令：`x`

这个命令可以检查指定内存范围的内容。

示例6.2.12 我们可以检查`main`的原始内容：

```bash
(gdb) x main
```
  
```text
0x804840b <main>: 0x04244c8d
```

默认情况下，不带任何参数的这个命令只打印单个内存地址的内容。示例中，就只打印了`main`的起始内存地址。

示例6.2.13 有了格式参数，这个命令可以以特定的格式打印某个范围内的内存：

```bash
(gdb) x/20b main
```

```text
0x804840b <main>:     0x8d  0x4c  0x24  0x04  0x83  0xe4  0xf0  0xff
0x8048413 <main+8>:   0x71  0xfc  0x55  0x89  0xe5  0x51  0x83  0xec
0x804841b <main+16>:  0x04  0x83  0xec  0x0c
```

`/20b main`参数意思是命令从`main`的位置开始打印20个字节

格式参数的一般形式是：`/<repeated count><format letter>`

如果没有提供重复计数，`gdb`默认提供的计数为`1`。格式字母可以是下列之一：

| 字母 | 描述 |
|-----|----- |
| o | 以八进制格式打印内存里的内容。 |
| x | 以十六进制格式打印内存里的内容。 |
| d | 以十进制格式打印内存里的内容。 |
| u | 以无符号十进制格式打印内存里的内容。 |
| t | 以二进制格式打印内存里的内容。 |
| f | 以浮点格式打印内存里的内容。 |
| a | 以内存地址格式打印内存里的内容。 |
| i | 以一系列汇编指令的格式打印内存里的内容。 |
| c | 以ASCII码数组格式打印内存里的内容。 |
| s | 以字符串格式打印内存里的内容。 |

在不同的场景下，某些格式要比其他格式更有优势。如果一个内存区域包含的都是浮点数字，那么使用`f`格式要比把数字看成分离的单字节十六进制数字要好得多。

### 6.2.7 命令：`print/p`

检查原始内存是很有用的，然而通常情况下，最好可以有更容易被人阅读的输出。这个命令就是用来完成这个任务的：它可以完美输出一个表达式。表达式可以是一个全局变量，也可以是当前堆栈框架中的一个局部变量，还可以是一个函数、一个寄存器、一个数字等等。

## 6.3 程序的运行时检查

调试器的主要用途是在程序运行时检查它的状态。`gdb`提供了一套有用的命令来检索有用的运行时信息。

### 6.3.1 命令：`run`

这个命令启动目标程序。

示例6.3.1 运行`hello`程序：

```bash
(gdb) r
```

```text
Starting program: /tmp/hello 
Hello World!
[Inferior 1 (process 1002) exited normally]
```

程序成功运行，并打印出了“Hello World”的信息。然而，如果`gdb`能做的只是运行一个程序，那就没有什么用了。

### 6.3.2 命令：`break/b`

这个命令在高阶源代码的某个位置设置一个断点。当`gdb`运行到断点标记的特定位置时，它会停止执行，以便程序员检查程序的当前状态。

示例6.3.2 断点可以设置在编辑器所显示的行上。假设我们想在程序的第3行设置一个断点，也就是`main`函数的开始位置：

```c
1 #include <stdio.h>
2 
3 int main(int argc, char *argv[])
4 {
5     printf("Hello World!\n");
6     return 0;
7 }
```

运行程序，`gdb`不是从头到尾运行，而是在第3行停下来：

```bash
  (gdb) b 3
```
  
```text
Breakpoint 1 at 0x400535: file hello.c, line 3.
```

```bash
(gdb) r
```

```bash
Starting program: /tmp/hello 
Breakpoint 1, main (argc=1, argv=0x7fffffffdfb8) at hello.c:5
5     printf("Hello World!\n");
```

断点在第3行，但`gdb`停在第5行。原因是第3行不包含代码，而是一个函数签名；`gdb`只在它能执行代码的地方停止。函数中的代码从第5行开始，即对`printf`的调用，所以`gdb`在那里停止。

示例6.3.3 代码的行号并不总是指定断点的可靠方法，因为源代码会改变。如果想要`gdb`总是停在`main`函数上该怎么办？在这种情况下，更好的方法是直接使用函数名：

```bash
b main
```

于是无论源代码如何变化，`gdb`总是停在主函数上。  

示例6.3.4 有时，调试程序没有包含调试信息，或者`gdb`正在调试汇编代码。在这种情况下，可以指定一个内存地址作为停止点。为了得到这个函数地址，可以使用`print`命令:

```bash
(gdb) print main
```

```text
$3 = {int (int, char **)} 0x400526 <main>
```

知道了`main`的地址，我们可以很容易地用内存地址设置一个断点：

```bash
b *0x400526
```
  
示例6.3.5 `gdb`还可以在任何源文件中设置断点。假设`hello`程序不是由单个文件而是由许多文件组成，例如`hello1.c`、`hello2.c`、`hello3.c`...在这种情况下，只需在行号前加上文件名：

```bash
b hello.c:3
```

示例6.3.6 也可以指定特定文件中的函数名：

```bash
b hello.c:main
```

### 6.3.3 命令：`next/n`

这个命令执行当前行，然后在下一行停止。如果当前行是一个函数调用，则单步执行它。

示例6.3.7 在`main`处设置断点后，运行一个程序并在第一个`printf`处停止:

```bash
(gdb) r
```

```text
Starting program: /tmp/hello 
Breakpoint 1, main (argc=1, argv=0x7fffffffdfb8) at hello.c:5
5     printf("Hello World!\n");
```

接着，为了前进到下一个语句，我们使用`next`命令：

```bash
(gdb) n
```

```text
Hello World!
6     return 0;
```

在输出中，第一行显示了执行第5行后产生的输出；然后，下一行显示了`gdb`当前停止的地方，也就是第6行。

181

  Command: step/s

This command executes the current line and stops at the next 
line. When the current line is a function call, steps into it to 
the first next line in the called function.

Suppose we have a new function add[footnote:
Why should we add a new function and function call instead of 
using the existing printf call? Stepping into shared library 
functions is tricky because to make debugging works, the debug 
info must be installed and loaded. It is not worth the trouble 
for demonstrating this simple command.
]:

  #include <stdio.h>



int add(int a, int b) {

	return a + b;

}



int main(int argc, char *argv[])

{

	add(1, 2);

    printf("Hello World!\n");

    return 0;

}

  If step command is used instead of next on the function call 
  printf, gdb steps inside the function:

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Breakpoint 1, main (argc=1, argv=0xffffd154) at hello.c:11

  11	    add(1, 2);

  

  

  (gdb) s

  

  

  add (a=1, b=2) at hello.c:6

  6	    return a + b;

  

  After executing the command s, gdb stepped into the add 
  function where the first statement is a return.

  Command: ni

At the core, gdb operates on assembly instruction. Source line by 
line debugging is simply an enhancement to make it friendlier for 
programmers. Each statement in C translates to one or more 
assembly instruction, as shown with objdump and disassemble 
command. With the debug info available, gdb knows how many 
instructions belong to one line of high-level code; line by line 
debugging is just a execution of assembly instructions of a line 
when moving from the current line to the next.

This command executes the one assembly instruction belongs to the 
current line. Until all assembly instructions of the current line 
are executed, gdb will not move to the next line. If the current 
instruction is a call, step over it to the next instruction.

When breakpoint is on the printf call and ni is used, it steps 
through each assembly instruction:

  

  (gdb) disassemble /s main

  

  

  Dump of assembler code for function main:

  hello.c:

  4	{

     0x0804840b <+0>:	 lea    ecx,[esp+0x4]

     0x0804840f <+4>:	 and    esp,0xfffffff0

     0x08048412 <+7>:	 push   DWORD PTR [ecx-0x4]

     0x08048415 <+10>:	push   ebp

     0x08048416 <+11>:	mov    ebp,esp

     0x08048418 <+13>:	push   ecx

     0x08048419 <+14>:	sub    esp,0x4

  5	    printf("Hello World!\n");

     0x0804841c <+17>:	sub    esp,0xc

     0x0804841f <+20>:	push   0x80484c0

     0x08048424 <+25>:	call   0x80482e0 <puts@plt>

     0x08048429 <+30>:	add    esp,0x10

  6	    return 0;

  => 0x0804842c <+33>:	mov    eax,0x0

  7	}

     0x08048431 <+38>:	mov    ecx,DWORD PTR [ebp-0x4]

     0x08048434 <+41>:	leave  

     0x08048435 <+42>:	lea    esp,[ecx-0x4]

     0x08048438 <+45>:	ret    

  End of assembler dump.

  

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Breakpoint 1, main (argc=1, argv=0xffffd154) at hello.c:5

  5	    printf("Hello World!\n");

  

  

  (gdb) ni

  

  

  0x0804841f	5	    printf("Hello World!\n");

  

  

  (gdb) ni

  

  

  0x08048424	5	    printf("Hello World!\n");

  

  

  (gdb) ni

  

  

  Hello World!

  0x08048429	5	    printf("Hello World!\n");

  

  

  (gdb)

  

  

  6	    return 0;

  

  Upon entering ni, gdb executes current instruction and display 
  the next instruction. That's why from the output, gdb only 
  displays 3 addresses: 0x0804841f, 0x08048424 and 0x08048429. 
  The instruction at 0x0804841c, which is the first instruction 
  of printf, is not displayed because it is the first instruction 
  that gdb stopped at. Assume that gdb stopped at the first 
  instruction of printf at 0x0804841c, the current instruction 
  can be displayed using x command:

  

  (gdb) x/i $eip

  

  

  => 0x804841c <main+17>: sub    esp,0xc

  

  Command: si

Similar to ni, this command executes the current assembly 
instruction belongs to the current line. But if the current 
instruction is a call, step into it to the first next instruction 
in the called function.

Recall that the assembly code generated from printf contains a 
call instruction:

  

  (gdb) disassemble /s main

  

  

  Dump of assembler code for function main:

  hello.c:

  4	{

     0x0804840b <+0>:	lea    ecx,[esp+0x4]

     0x0804840f <+4>:	and    esp,0xfffffff0

     0x08048412 <+7>:	push   DWORD PTR [ecx-0x4]

     0x08048415 <+10>:	push   ebp

     0x08048416 <+11>:	mov    ebp,esp

     0x08048418 <+13>:	push   ecx

     0x08048419 <+14>:	sub    esp,0x4

  5	    printf("Hello World!\n");

     0x0804841c <+17>:	sub    esp,0xc

     0x0804841f <+20>:	push   0x80484c0

     0x08048424 <+25>:	call   0x80482e0 <puts@plt>

     0x08048429 <+30>:	add    esp,0x10

  6	    return 0;

  => 0x0804842c <+33>:	mov    eax,0x0

  7	}

     0x08048431 <+38>:	mov    ecx,DWORD PTR [ebp-0x4]

     0x08048434 <+41>:	leave  

     0x08048435 <+42>:	lea    esp,[ecx-0x4]

     0x08048438 <+45>:	ret    

  End of assembler dump.

  

  We try instruction by instruction stepping again, but this time 
  by running si at 0x08048424, where call resides:

  

  (gdb) si

  

  

  0x0804841f	5	        printf("Hello World!\n");

  

  

  (gdb) si

  

  

  0x08048424	5	        printf("Hello World!\n");

  

  

  (gdb) x/i $eip

  

  

  => 0x8048424 <main+25>:	call   0x80482e0 <puts@plt>

  

  

  (gdb) si

  

  

  0x080482e0 in puts@plt ()

  

  The next instruction right after 0x8048424 is the first 
  instruction at 0x080482e0 in puts function. In other words, gdb 
  stepped into puts instead of stepping over it.

  Command: until

This command executes until the next line is greater than the 
current line.

Suppose we have a function that execute a long loop:

  #include <stdio.h>



int add1000() {

    int total = 0;



    for (int i = 0; i < 1000; ++i){

        total += i;

    }



    printf("Done adding!\n");



    return total;

}



int main(int argc, char *argv[])

{

    add1000(1, 2);

    printf("Hello World!\n");

    return 0;

}

  Using next command, we need to press 1000 times for finishing 
  the loop. Instead, a faster way is to use until: 

  

  (gdb) b add1000

  

  

  Breakpoint 1 at 0x8048411: file hello.c, line 4.

  

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Breakpoint 1, add1000 () at hello.c:4

  4	    int total = 0;

  

  

  (gdb) until

  

  

  5	    for (int i = 0; i < 1000; ++i){

  

  

  (gdb) until

  

  

  6	        total += i;

  

  

  (gdb) until

  

  

  5	    for (int i = 0; i < 1000; ++i){

  

  

  (gdb) until

  

  

  8	    printf("Done adding!\n");

  

  Executing the first until, gdb stopped at line 5 since line 5 
  is greater than line 4. 

  Executing the second until, gdb stopped at line 6 since line 6 
  is greater than line 5.

  Executing the third until, gdb stopped at line 5 since the loop 
  still continues. Because line 5 is less than line 6, with the 
  fourth until, gdb kept executing until it does not go back to 
  line 5 anymore and stopped at line 8. This is a great way to 
  skip over loop in the middle, instead of setting unneeded 
  breakpoint.

  until can be supplied with an argument to explicitly execute to 
  a specific line:

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Breakpoint 1, add1000 () at hello.c:4

  4	    int total = 0;

  

  

  (gdb) until 8

  

  

  add1000 () at hello.c:8

  8	    printf("Done adding!\n");

  

  Command: finish

This command executes until the end of a function and displays 
the return value. finish is actually just a more convenient 
version of until.

Using the add1000 function from the previous example and use 
finish instead of until:

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Breakpoint 1, add1000 () at hello.c:4

  4	    int total = 0;

  

  

  (gdb) finish

  

  

  Run till exit from #0  add1000 () at hello.c:4

  Done adding!

  0x08048466 in main (argc=1, argv=0xffffd154) at hello.c:15

  15	    add1000(1, 2);

  Value returned is $1 = 499500

  

  Command: bt

This command prints the backtrace of all stack frames. A [margin:
backtrace
]backtracebacktrace is a list of currently active functions:

Suppose we have a chain of function calls:

  void d(int d) { };

void c(int c) { d(0); }

void b(int b) { c(1); }

void a(int a) { b(2); }



int main(int argc, char *argv[])

{

    a(3);

    return 0;

}

  bt can visualize such a chain in action:

  

  (gdb) b a

  

  

  Breakpoint 1 at 0x8048404: file hello.c, line 9.

  

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Breakpoint 1, a (a=3) at hello.c:9

  9	void a(int a) { b(2); }

  

  

  (gdb) s

  

  

  b (b=2) at hello.c:7

  7	void b(int b) { c(1); }

  

  

  (gdb) s

  

  

  c (c=1) at hello.c:5

  5	void c(int c) { d(0); }

  

  

  (gdb) s

  

  

  d (d=0) at hello.c:3

  3	void d(int d) { };

  

  

  (gdb) bt

  

  

  #0  d (d=0) at hello.c:3

  #1  0x080483eb in c (c=1) at hello.c:5

  #2  0x080483fb in b (b=2) at hello.c:7

  #3  0x0804840b in a (a=3) at hello.c:9

  #4  0x0804841b in main (argc=1, argv=0xffffd154) at hello.c:13

  

  Most-recent calls are placed on top and least-recent calls are 
  near the bottom. In this case, d is the most current active 
  function, so it has the index 0. Next is c, the 2[superscript:nd] active function, has the index 1 and so on with function b, 
  function a, and finally function main at the bottom, the 
  least-recent function. That is how we read a backtrace.

  Command: up

This command goes up one frame earlier the current frame.

Instead of staying in d function, we can go up to c function and 
look at its state:

  

  (gdb) bt

  

  

  #0  d (d=0) at hello.c:3

  #1  0x080483eb in c (c=1) at hello.c:5

  #2  0x080483fb in b (b=2) at hello.c:7

  #3  0x0804840b in a (a=3) at hello.c:9

  #4  0x0804841b in main (argc=1, argv=0xffffd154) at hello.c:13

  

  

  (gdb) up

  

  

  #1  0x080483eb in c (c=1) at hello.c:3

  3	void b(int b) { c(1); }

  

  The output displays the current frame is moved to c and where 
  the call to c is made, which is in function b at line 3.

  Command: down

Similar to up, this command goes down one frame later then the 
current frame.

After inspecting c function, we can go back to d:

  

  (gdb) bt

  

  

  #0  d (d=0) at hello.c:3

  #1  0x080483eb in c (c=1) at hello.c:5

  #2  0x080483fb in b (b=2) at hello.c:7

  #3  0x0804840b in a (a=3) at hello.c:9

  #4  0x0804841b in main (argc=1, argv=0xffffd154) at hello.c:13

  

  

  (gdb) up

  

  

  #1  0x080483eb in c (c=1) at hello.c:3

  3	void b(int b) { c(1); }

  

  

  (gdb) down

  

  

  #0  d (d=0) at hello.c:1

  1	void d(int d) { };

  

  Command: info registers

This command lists the current values in commonly used registers. 
This command is useful when debugging assembly and operating 
system code, as we can inspect the current state of the machine.

Executing the command, we can see the commonly used registers:

  

  (gdb) info registers 

  

  

  eax            0xf7faddbc	-134554180

  ecx            0xffffd0c0	-12096

  edx            0xffffd0e4	-12060

  ebx            0x0	0

  esp            0xffffd0a0	0xffffd0a0

  ebp            0xffffd0a8	0xffffd0a8

  esi            0xf7fac000	-134561792

  edi            0xf7fac000	-134561792

  eip            0x804841c	0x804841c <main+17>

  eflags         0x286	[ PF SF IF ]

  cs             0x23	35

  ss             0x2b	43

  ds             0x2b	43

  es             0x2b	43

  fs             0x0	0

  gs             0x63	99

  

  The above registers suffice for writing our operating system in 
  later part.

  How debuggers work: A brief introduction

  How breakpoints work

When a programmer places a breakpoint somewhere in his code, what 
actually happens is that the first opcode of the first 
instruction of a statement is replaced with another instruction, 
int 3 with opcode CCh:

[float Figure:
[Figure 0.17:
Opcode replacement, with int 3
]

     
+-----+-----+---------+                +-----+-----+----+
| 83  | ec  |   0c    |  \rightarrow
  | cc  | ec  | 0c |
+-----+-----+---------+                +-----+-----+----+
+---------------------+                +----------------+
|     sub esp,0x4     |                |     int 3      |
+---------------------+                +----------------+
     
]

int 3 only costs a single byte, making it efficient for 
debugging. When int 3 instruction is executed, the operating 
system calls its breakpoint interrupt handler. The handler then 
checks what process reaches a breakpoint, pauses it and notifies 
the debugger it has paused a debugged process. The debugged 
process is only paused and that means a debugger is free to 
inspect its internal state, like a surgeon operates on an 
anesthetic patient. Then, the debugger replaces the int 3 opcode 
with the original opcode and executes the original instruction 
normally.

[float Figure:
[Figure 0.18:
Restore the original opcode, after int 3 was executed
]

     
+-----+-----+----+                +-----+-----+---------+
| cc  | ec  | 0c |  \rightarrow
  | 83  | ec  |   0c    |
+-----+-----+----+                +-----+-----+---------+
+----------------+                +---------------------+
|     int 3      |                |     sub esp,0x4     |
+----------------+                +---------------------+
     
]

It is simple to see int 3 in action. First, we add an int 3 
instruction where we need gdb to stop:

  #include <stdio.h>



int main(int argc, char *argv[])

{

    asm("int 3");

    printf("Hello World\n");

    return 0;

}

  int 3 precedes printf, so gdb is expected to stop at printf. 
  Next, we compile with debug enable and with Intel syntax:

  

  $ gcc -masm=intel -m32 -g hello.c -o hello

  

  Finally, start gdb:

  

  $ gdb hello

  

  Running without setting any breakpoint, gdb stops at printf 
  call, as expected:

  

  (gdb) r

  

  

  Starting program: /tmp/hello 

  Program received signal SIGTRAP, Trace/breakpoint trap.

  main (argc=1, argv=0xffffd154) at hello.c:6

  6	    printf("Hello World\n");

  

  The blue text indicates that gdb encountered a breakpoint, and 
  indeed it stopped at the right place: the printf call, where 
  int 3 preceded it.

  Single stepping

When breakpoint is implemented, it is easy to implement single 
stepping: a debugger simply places another int 3 opcode in the 
next instruction. So, when a programmer sets a breakpoint at an 
instruction, the next instruction is automatically set by the 
debugger, thus enable instruction by instruction debugging. 
Similarly, source line by line debugging is just the placements 
of the very first opcodes in the two statements with two int 3 
opcodes.

  How a debugger understands high level source code

DWARF is a debugging file format used by many compilers and 
debuggers to support source level debugging. DWARF contains 
information that maps between entities in the executable binary 
with the source files. A program entity can either be data or 
code. A DIE, or [margin:
Debugging Information Entry
]Debugging Information EntryDebugging Information Entry, is a 
description of a program entity. A DIE consists of a 
tag, which specifies the entity 
that the DIE describes, and a list of  attributes that describes 
the entity. Of all the attributes, these two attributes enables 
source-level debugging: 

• Where the entity appears in the source files: which file and 
  which line the entity appears.

• Where the entity appears in the executable binary: in which 
  memory address the entity is loaded at runtime. With the 
  precise address, gdb can retrieve correct value for a data 
  entity, or place a correct breakpoint and stop accordingly for 
  a code entity. Without the information of these addresses, gdb 
  would not know where the entities are to inspect them.




+---------------------------------------------------------------------------------------------------------------------------------------------------+                          +------------------------------------------------------------------+
| hello.c                                                                                                                                           |                          | DIE                                                              |
+---------------------------------------------------------------------------------------------------------------------------------------------------+                          +------------------------------------------------------------------+
+--------------------------------------------------------------+------------------------------------------------------------------------------------+                          +------------------------------------------------------------------+
|   Line 1

  Line 2

\Rightarrow
 Line 3

  Line 5

  Line 6  | #include <stdio.h>

 

int main(int argc, char *argv[])

..........

..........

  |   

 

  \rightarrow
    | ....

....

main in hello.c is at 0x804840b in hello

....

.... |
+--------------------------------------------------------------+------------------------------------------------------------------------------------+                          +------------------------------------------------------------------+
                                                                                                                                                                                                                                                   
                                                                                                                                                                                                     \downarrow
\uparrow
                          
                                                                                                                                                                                                                                                   
                                                                                                                                                                               +------------------------------------------------------------------+
                                                                                                                                                                               | hello (at 0x804840b)                                             |
                                                                                                                                                                               +------------------------------------------------------------------+
                                                                                                                                                                               +------------------------------------------------------------------+
                                                                                                                                                                               | ...8d 4c 24 04 83 e4 f0 ff 71 fc ....                            |
                                                                                                                                                                               +------------------------------------------------------------------+






In addition to DIEs, another binary-to-source mapping is the line 
number table. The line number table maps between a line in the 
source code and at which memory address is the start of the line 
in the executable binary.

In sum, to successfully enable source-level debugging, a debugger 
needs to know the precise location of the source files and the 
load addresses at runtime. Address matching, between the image 
layout of the ELF binary and the address where it is loaded, is 
extremely important since debug information relies on correct 
loading address at runtime. That is, it assumes the addresses as 
recorded in the binary image at compile-time the same as at 
runtime e.g. if the load address for .text section is recorded in 
the executable binary at 0x800000, then when the binary actually 
runs, .text should really be loaded at 0x800000 for gdb to be 
able to correctly match running instructions with high-level code 
statement. Address mismatching makes debug information useless, 
as actual code at one address is displayed as code at another 
address. Without this knowledge, we will not be able to build an 
operating system that can be debugged with gdb.

When an executable binary contains debug info, readelf can 
display such information in a readable format. Using the good old 
hello world program:

  #include <stdio.h>



int main(int argc, char *argv[])

{

    printf("Hello World\n");



    return 0;

}

  and compile with debug info:

  

  $ gcc -m32 -g hello.c -o hello

  

  With the binary ready, we can look at the line number table 
  with the command:

  

  $ readlelf -wL hello

  

  -w option prints all the debug information. In combination with 
  its sub-option, only specific information is displayed. For 
  example, with -L, only the line number table is displayed:

  

  Decoded dump of debug contents of section .debug_line:

  CU: hello.c:

  File name                            Line number    Starting 
  address

  hello.c                                        6           
  0x804840b

  hello.c                                        7           
  0x804841c

  hello.c                                        9           
  0x804842c

  hello.c                                       10           
  0x8048431

  

  From the above output:

  CU shorts for Compilation Unit, a 
    separately compiled source file. In the example, we only have 
    one file, hello.c.

  File name displays the filename of the current compilation 
    unit.

  Line number is the line number in the source file of which the 
    line is not an empty line. In the example, line 8 is an empty 
    line, so it does not appear.

  Starting address is the memory address where the line actually 
    starts in the executable binary. 

  With such crystal clear information, this is how gdb is able to 
  set a breakpoint on a line easily. For placing breakpoints on 
  variables and functions, it is time to look at the DIEs. To get 
  the DIEs information from an executable binary, run the 
  command:

  

  $ readlelf -wi hello

  

  -wi option lists all the DIE entries. This is one typical DIE 
  entry:

   <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)

      <c>   DW_AT_producer    : (indirect string, offset: 0xe): 
  GNU C11 5.4.0 20160609 -masm=intel -m32 -mtune=generic 
  -march=i686 -g -fstack-protector-strong

      <10>   DW_AT_language    : 12	(ANSI C99)

      <11>   DW_AT_name        : (indirect string, offset: 0xbe): 
  hello.c

      <15>   DW_AT_comp_dir    : (indirect string, offset: 0x97): 
  /tmp

      <19>   DW_AT_low_pc      : 0x804840b

      <1d>   DW_AT_high_pc     : 0x2e

      <21>   DW_AT_stmt_list   : 0x0

  Red This left-most number indicates the current nesting level 
    of a DIE entry. 0 is the outer-most level DIE with its entity 
    is the compilation unit. This means subsequent DIE entries 
    with higher nesting level are all the children of this tag, 
    the compilation unit. It makes sense, as all the entities 
    must originate from a source file.

  Blue These numbers in hex format indicate the offsets into 
    .debug_info section. Each meaningful information is displayed 
    along with its offset. When an attribute references to 
    another attribute, the offset is used to precisely identify 
    the referenced attribute.

  Green These names with DW_AT_ prefix are the attributes 
    attached to a DIE that describe an entity. Notable 
    attributes:

    DW_AT_name

    DW_AT_comp_dir The filename of the compilation unit and the 
      directory where compilation occurred. Without the filename 
      and the path, gdb would not be able to display the 
      high-level source, despite the availability of the debug 
      info. Debug info only contains the mapping between source 
      and binary, not the source code itself.

    DW_AT_low_pc

    DW_AT_high_pc The start and end of the current entity, which 
      is the compilation unit, in the executable binary. The 
      value in DW_AT_low_pc is the starting address. 
      DW_AT_high_pc is the size of the compilation unit, when 
      adding up to DW_AT_low_pc results in the end address of the 
      entity. In this example, code compiled from hello.c starts 
      at 0x804840b and end at \mathtt{0x804840b+0x2e=0x8048439}
. 
      To really make sure, we verify with objdump:

      

      int main(int argc, char *argv[])

      {

       804840b:       8d 4c 24 04             lea    
      ecx,[esp+0x4]

       804840f:       83 e4 f0                and    
      esp,0xfffffff0

       8048412:       ff 71 fc                push   DWORD PTR 
      [ecx-0x4]

       8048415:       55                      push   ebp

       8048416:       89 e5                   mov    ebp,esp

       8048418:       51                      push   ecx

       8048419:       83 ec 04                sub    esp,0x4

          printf("Hello World\n");

       804841c:       83 ec 0c                sub    esp,0xc

       804841f:       68 c0 84 04 08          push   0x80484c0

       8048424:       e8 b7 fe ff ff          call   80482e0 
      <puts@plt>

       8048429:       83 c4 10                add    esp,0x10

          return 0;

       804842c:       b8 00 00 00 00          mov    eax,0x0

      }

       8048431:       8b 4d fc                mov    ecx,DWORD 
      PTR [ebp-0x4]

       8048434:       c9                      leave  

       8048435:       8d 61 fc                lea    
      esp,[ecx-0x4]

       8048438:       c3                      ret    

       8048439:       66 90                   xchg   ax,ax

       804843b:       66 90                   xchg   ax,ax

       804843d:       66 90                   xchg   ax,ax

       804843f:       90                      nop

      

      It is true: main starts at 804840b and end at 8048439, 
      right after the ret instruction at 8048438. The 
      instructions after 8048439 are just padding bytes inserted 
      by gcc for alignment, which do not belong to main. Note 
      that the output from objdump shows much more code past 
      main. It is not counted, as the code is outside of hello.c, 
      added by gcc for the operating system. hello.c contains 
      only one function: main and this is why hello.c also starts 
      and ends the same as main.

  Pink This number displays the abbreviation form of a tag. An 
    abbreviation is the form of a DIE. When debug info is 
    displayed with -wi, the DIEs are displayed with their values. 
    -wa option shows abbreviations in the .debug_abbrev section:

    

    Contents of the .debug_abbrev section:

      Number TAG (0x0)

       1      DW_TAG_compile_unit    [has children]

        DW_AT_producer     DW_FORM_strp

        DW_AT_language     DW_FORM_data1

        DW_AT_name         DW_FORM_strp

        DW_AT_comp_dir     DW_FORM_strp

        DW_AT_low_pc       DW_FORM_addr

        DW_AT_high_pc      DW_FORM_data4

        DW_AT_stmt_list    DW_FORM_sec_offset

        DW_AT value: 0     DW_FORM value: 0

    .... more abbreviations ....

    

    The output is similar to a DIE output, with only attribute 
    names and without any value. We can also say an abbreviation 
    is a type of a DIE, as an abbreviation represents the 
    structure of a particular DIE. Many DIEs share the same 
    abbreviation, or structure, thus they are of the same type. 
    An abbreviation number specifies which type a DIE is in the 
    abbreviation table above. Abbreviations improve encoding 
    efficiency (reduce binary size) because each DIE needs not to 
    carry their structure information as pairs of attribute-value[footnote:
For example, data format such as YAML or JSON encodes its 
attribute names along with its values. This simplifies encoding, 
but with overhead.
], but simply refers to an abbreviation for correct decoding.

  Here are all the DIEs of hello represented as a tree:

  

  <Graphics file: C:/Users/Tu Do/os01/book_src/images/06/dwarf_tree.svg>
  <dwarf_tree>

  

In the figure [dwarf_tree], DW_TAG_subprogram represents a 
function such as main. Its children are the DIEs of argc and 
argv. With such precise information, matching source to binary is 
an easy job for gdb.

If more than one compilation units exist in an executable binary, 
the DIE entries are sorted according to the compilation order 
from gcc. For example, suppose we have another test.c source file[footnote:
It can contain anything. Just a sample file.
] and compile it together with hello:



$ gcc -masm=intel -m32 -g test.c hello.c -o hello



Then, the all DIE entries in test.c are displayed before the DIE 
entries in hello.c:

<0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)

      <c>   DW_AT_producer    : (indirect string, offset: 0x0): 
  GNU C11 5.4.0 20160609 

  -masm=intel -m32 -mtune=generic -march=i686 -g 
  -fstack-protector-strong

      <10>   DW_AT_language    : 12       (ANSI C99)

      <11>   DW_AT_name        : (indirect string, offset: 0x64): 
  test.c

      <15>   DW_AT_comp_dir    : (indirect string, offset: 0x5f): 
  /tmp

      <19>   DW_AT_low_pc      : 0x804840b

      <1d>   DW_AT_high_pc     : 0x6

      <21>   DW_AT_stmt_list   : 0x0

   <1><25>: Abbrev Number: 2 (DW_TAG_subprogram)

      <26>   DW_AT_external    : 1

      <26>   DW_AT_name        : bar

      <2a>   DW_AT_decl_file   : 1

      <2b>   DW_AT_decl_line   : 1

      <2c>   DW_AT_low_pc      : 0x804840b

      <30>   DW_AT_high_pc     : 0x6

      <34>   DW_AT_frame_base  : 1 byte block: 9c         
  (DW_OP_call_frame_cfa)

      <36>   DW_AT_GNU_all_call_sites: 1

  

  ....after all DIEs in test.c listed....