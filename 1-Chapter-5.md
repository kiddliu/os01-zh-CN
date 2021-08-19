# 第五章 解剖一个程序

每个程序都是由代码和数据组成，且只由这两部分组成。然而，如果一个程序纯粹由代码和数据组成，那么从操作系统（以及人类）的角度来看，就不清楚在一个程序中，二进制的哪一部分是程序，哪一部分只是原始数据，程序是从什么地方开始执行，哪些区域的内存应该被保护，哪些可以自由修改。出于这个原因，每个程序都附带额外的元数据，以便与操作系统理解如何处理这个程序。

当源文件被编译时，生成的机器代码被存储到*对象文件*中，是纯二进制的数据块。一或多个对象文件可以被组合起来，生成*可执行的二进制文件*，它是一个可以在操作系统中运行的完整程序。

`readelf`是一个能够识别并显示二进制文件的ELF元数据的程序，无论这个二进制文件只是对象文件还是可执行程序。`ELF`，即可执行与可链接格式，是位于可执行文件起始位置的内容，旨在为操作系统加载该可执行文件到内存并执行提供必要的信息。ELF与一本书的目录类似。在一本书中，目录列出了主要章节的页码、子章节，有时为了便于查找甚至还提供数字与表格。同样，ELF列出了用作代码和数据的各个节，以及每个符号的内存地址和其他信息。

一个`ELF`二进制文件由以下部分组成：

* *`ELF`头*：可执行文件的首个节，描述了文件的组织结构。
* *程序标头表*：是一个固定大小结构体的数组，描述了可执行文件的每个段。
* *节标头表*：是一个固定大小结构体的数组，描述可执行文件的每个节。
* *段、节*：段与节是`ELF`二进制文件的主要内容，根据不同的用途分成了代码块和数据块。

*段*是由零个或多个节组成，在运行时直接被操作系统加载。

*节*是一个二进制的块，可以是：

* 在程序运行时可在内存中使用的实际程序代码与数据
* 只在链接过程中使用的描述其他节的元数据，最终并不出现在最终的可执行文件中。

链接器使用节来构建段。

![图5.0.1 ELF - 链接视角与可执行文件视角（来源：Wikipedia）](images/05/Elf-layout--en.png)

稍后我们将使用`GCC`将我们的内核编译成ELF可执行文件，并通过使用链接器脚本明确指定段的创建方式以及它们在内存中的加载位置，这里的链接器脚本是一个文本文件，指示链接器应该如何生成二进制文件。现在，我们将详细研究ELF可执行文件的结构。



  Reference documents: 

The [margin:
ELF specification
]ELF specification is bundled as a man page in Linux:



$ man elf



It is a useful resource to understand and implement ELF. However, 
it will be much easier to use after you finish this chapter, as 
the specification mixes implementation details in it.

The default specification is a generic one, in which every ELF 
implementation follows. However, each platform provides extra 
features unique to it. The ELF specification for x86 is currently 
maintained on Github by H.J. Lu: https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI
. 

Platform-dependent details are referred to as “processor specific”
 in the generic ELF specification. We will not explore these 
details, but study the generic details, which are enough for 
crafting an ELF binary image for our operating system.

  ELF header

To see the information of an ELF header:



$ readelf -h hello



The output:



ELF Header:

  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00

  Class:                             ELF64   

  Data:                              2's complement, little 
endian

  Version:                           1 (current)

  OS/ABI:                            UNIX - System V

  ABI Version:                       0

  Type:                              EXEC (Executable file)

  Machine:                           Advanced Micro Devices 
X86-64

  Version:                           0x1

  Entry point address:               0x400430

  Start of program headers:          64 (bytes into file)

  Start of section headers:          6648 (bytes into file)

  Flags:                             0x0

  Size of this header:               64 (bytes)

  Size of program headers:           56 (bytes)

  Number of program headers:         9

  Size of section headers:           64 (bytes)

  Number of section headers:         31

Section header string table index: 28



Let's go through each field:

  Magic

  Displays the raw bytes that uniquely addresses a file is an ELF 
  executable binary. Each byte gives a brief information.

  In the example, we have the following magic bytes:

  

  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00

  

  Examine byte by byte:

   
  Byte                       Description                                                                                                                                                                     
  -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    7f 45 4c 46                Predefined values. The first byte is always 7F, the remaining 3 
bytes represent the string “ELF”.                                                                              
                                                                                                                                                                                                               
    02                         See Class field below.                                                                                                                                                          
                                                                                                                                                                                                               
    01                         See Data field below.                                                                                                                                                           
                                                                                                                                                                                                               
    01                         See Version field below.                                                                                                                                                        
                                                                                                                                                                                                               
    00                         See OS/ABI field below.                                                                                                                                                         
                                                                                                                                                                                                               
    00 00 00 00 00 00 00 00    Padding bytes. These bytes are unused and are always set to 0. 
Padding bytes are added for proper alignment, and is reserved for 
future use when more information is needed.  
  

  Class

  A byte in Magic field. It specifies the class or capacity of a 
  file. 

  Possible values:

   
  Value     Description    
  ---------------------------
      0      Invalid class   
      1      32-bit objects  
      2      64-bit objects  
  

  Data

  A byte in Magic field. It specifies the data encoding of the 
  processor-specific data in the object file.

  Possible values:

   
  Value    Description                    
  ------------------------------------------
      0      Invalid data encoding          
      1      Little endian, 2's complement  
      2      Big endian, 2's complement     
  

  Version

  A byte in Magic. It specifies the ELF header version number.

  Possible values:

   
  Value    Description      
  ----------------------------
      0      Invalid version  
      1      Current version  
  

  OS/ABI

  A byte in Magic field. It specifies the target operating system 
  ABI. Originally, it was a padding byte.

  Possible values: Refer to the latest ABI document, as it is a 
  long list of different operating systems.

  Type

  Identifies the object file type.

   
  Value     Description                      
            -----------------------------------
  ---------------------------------------------
      0       No file type                     
      1       Relocatable file                 
      2       Executable file                  
      3       Shared object file               
      4       Core file                        
    0xff00    Processor specific, lower bound  
    0xffff    Processor specific, upper bound  
  

  The values from 0xff00 to 0xffff are reserved for a processor 
  to define additional file types meaningful to it.

  Machine

  Specifies the required architecture value for an ELF file e.g. 
  x86_64, MIPS, SPARC, etc. In the example, the machine is of x86_64
   architecture.

  Possible values: Please refer to the latest ABI document, as it 
  is a long list of different architectures.

  Version

  Specifies the version number of the current object file (not 
  the version of the ELF header, as the above Version field 
  specified).

  Entry point address

  Specifies the memory address where the very first code to be 
  executed. The address of main function is the default in a 
  normal application program, but it can be any function by 
  explicitly specifying the function name to gcc. For the 
  operating system we are going to write, this is the single most 
  important field that we need to retrieve to bootstrap our 
  kernel, and everything else can be ignored.

  Start of program headers

  The offset of the program header table, in bytes. In the 
  example, this number is 64 bytes, which means the 65th byte, or 
  <start address> + 64, is the start address of the program 
  header table. That is, if a program is loaded at address 0x10000
   in memory, then the start address is 0x10000 (the very first 
  byte of Magic field, where the value 0x7f resides) and the 
  start address of program header table is 0x10000 + 0x40 = 0x10040
  .

  Start of section headers

  The offset of the section header table in bytes, similar to the 
  start of program headers. In the example, it is 6648 bytes into 
  file.

  Flags

  Hold processor-specific flags associated with the file. When 
  the program is loaded, in a x86 machine, EFLAGS register is set 
  according to this value. In the example, the value is 0x0, 
  which means EFLAGS register is in a clear state.

  Size of this header

  Specifies the total size of ELF header's size in bytes. In the 
  example, it is 64 bytes, which is equivalent to Start of 
  program headers. Note that these two numbers are not necessarily 
  equivalent, as program header table might be placed far away 
  from the ELF header. The only fixed component in the ELF 
  executable binary is the ELF header, which appears at the very 
  beginning of the file.

  Size of program headers

  Specifies the size of each program header in bytes. In the 
  example, it is 64 bytes.

  Number of program headers

  Specifies the total number of program headers. In the example, 
  the file has a total of 9 program headers.

  Size of section headers

  Specifies the size of each section header in bytes. In the 
  example, it is 64 bytes.

  Number of section headers

  Specifies the total number of section headers. In the example, 
  the file has a total of 31 section headers. In a section header 
  table, the first entry in the table is always an empty section.

  Section header string table index

  Specifies the index of the header in the section header table 
  that points to the section that holds all null-terminated 
  strings. In the example, the index is 28, which means it's the 
  28[superscript:th] entry of the table. 

  Section header table

As we know already, code and data compose a program. However, not 
all types of code and data have the same purpose. For that 
reason, instead of a big chunk of code and data, they are divided 
into smaller chunks, and each chunk must satisfy these conditions 
(according to gABI):

• Every section in an object file has exactly one section header 
  describing it. But, section headers may exist that do not have 
  a section.

• Each section occupies one contiguous (possibly empty) sequence 
  of bytes within a file. That means, there's no two regions of 
  bytes that are the same section.

• Sections in a file may not overlap. No byte in a file resides 
  in more than one section.

• An object file may have inactive space. The various headers and 
  the sections might not “cover” every byte in an object file. 
  The contents of the inactive data are unspecified.

To get all the headers from an executable binary e.g. hello, use 
the following command:



$ readelf -S hello



Here is a sample output (do not worry if you don't understand the 
output. Just skim to get your eyes familiar with it. We will 
dissect it soon enough):



There are 31 section headers, starting at offset 0x19c8:



Section Headers:

  [Nr] Name              Type             Address           
Offset

       Size              EntSize          Flags  Link  Info  
Align

  [ 0]                   NULL             0000000000000000  
00000000

       0000000000000000  0000000000000000           0     0     0

  [ 1] .interp           PROGBITS         0000000000400238  
00000238

       000000000000001c  0000000000000000   A       0     0     1

  [ 2] .note.ABI-tag     NOTE             0000000000400254  
00000254

       0000000000000020  0000000000000000   A       0     0     4

  [ 3] .note.gnu.build-i NOTE             0000000000400274  
00000274

       0000000000000024  0000000000000000   A       0     0     4

  [ 4] .gnu.hash         GNU_HASH         0000000000400298  
00000298

       000000000000001c  0000000000000000   A       5     0     8

  [ 5] .dynsym           DYNSYM           00000000004002b8  
000002b8

       0000000000000048  0000000000000018   A       6     1     8

  [ 6] .dynstr           STRTAB           0000000000400300  
00000300

       0000000000000038  0000000000000000   A       0     0     1

  [ 7] .gnu.version      VERSYM           0000000000400338  
00000338

       0000000000000006  0000000000000002   A       5     0     2

  [ 8] .gnu.version_r    VERNEED          0000000000400340  
00000340

       0000000000000020  0000000000000000   A       6     1     8

  [ 9] .rela.dyn         RELA             0000000000400360  
00000360

       0000000000000018  0000000000000018   A       5     0     8

  [10] .rela.plt         RELA             0000000000400378  
00000378

       0000000000000018  0000000000000018  AI       5    24     8

  [11] .init             PROGBITS         0000000000400390  
00000390

       000000000000001a  0000000000000000  AX       0     0     4

  [12] .plt              PROGBITS         00000000004003b0  
000003b0

       0000000000000020  0000000000000010  AX       0     0     
16

  [13] .plt.got          PROGBITS         00000000004003d0  
000003d0

       0000000000000008  0000000000000000  AX       0     0     8

  [14] .text             PROGBITS         00000000004003e0  
000003e0

       0000000000000192  0000000000000000  AX       0     0     
16

  [15] .fini             PROGBITS         0000000000400574  
00000574

       0000000000000009  0000000000000000  AX       0     0     4

  [16] .rodata           PROGBITS         0000000000400580  
00000580

       0000000000000004  0000000000000004  AM       0     0     4

  [17] .eh_frame_hdr     PROGBITS         0000000000400584  
00000584

       000000000000003c  0000000000000000   A       0     0     4

  [18] .eh_frame         PROGBITS         00000000004005c0  
000005c0

       0000000000000114  0000000000000000   A       0     0     8

  [19] .init_array       INIT_ARRAY       0000000000600e10  
00000e10

       0000000000000008  0000000000000000  WA       0     0     8

  [20] .fini_array       FINI_ARRAY       0000000000600e18  
00000e18

       0000000000000008  0000000000000000  WA       0     0     8

  [21] .jcr              PROGBITS         0000000000600e20  
00000e20

       0000000000000008  0000000000000000  WA       0     0     8

  [22] .dynamic          DYNAMIC          0000000000600e28  
00000e28

       00000000000001d0  0000000000000010  WA       6     0     8

  [23] .got              PROGBITS         0000000000600ff8  
00000ff8

       0000000000000008  0000000000000008  WA       0     0     8

  [24] .got.plt          PROGBITS         0000000000601000  
00001000

       0000000000000020  0000000000000008  WA       0     0     8

  [25] .data             PROGBITS         0000000000601020  
00001020

       0000000000000010  0000000000000000  WA       0     0     8

  [26] .bss              NOBITS           0000000000601030  
00001030

       0000000000000008  0000000000000000  WA       0     0     1

  [27] .comment          PROGBITS         0000000000000000  
00001030

       0000000000000034  0000000000000001  MS       0     0     1

  [28] .shstrtab         STRTAB           0000000000000000  
000018b6

       000000000000010c  0000000000000000           0     0     1

  [29] .symtab           SYMTAB           0000000000000000  
00001068

       0000000000000648  0000000000000018          30    47     8

  [30] .strtab           STRTAB           0000000000000000  
000016b0

       0000000000000206  0000000000000000           0     0     1

Key to Flags:

  W (write), A (alloc), X (execute), M (merge), S (strings), l 
(large)

  I (info), L (link order), G (group), T (TLS), E (exclude), x 
(unknown)

  O (extra OS processing required) o (OS specific), p (processor 
specific)



The first line:



There are 31 section headers, starting at offset 0x19c8



summarizes the total number of sections in the file, and where 
the address where it starts. Then, comes the listing section by 
section with the following header, is also the format of each 
section output:



[Nr] Name              Type             Address           Offset

     Size              EntSize          Flags  Link  Info  Align



Each section has two lines with different fields:

  Nr The index of each section.

  Name The name of each section.

  Type This field (in a section header) identifies the type of 
  each section. Types classify sections (similar to types in 
  programming languages are used by a compiler). 

  Address The starting virtual address of each section. Note that 
  the addresses are virtual only when a program runs in an OS 
  with support for virtual memory enabled. In our OS, since we 
  run on bare metal, the addresses will all be physical.

  Offset The offset of each section into a file. An [margin:
offset
]offsetoffset is a distance in bytes, from the first byte of a 
  file to the start of an object, such as a section or a segment 
  in the context of an ELF binary file.

  Size The size in bytes of each section.

  EntSize Some sections hold a table of fixed-size entries, such 
  as a symbol table. For such a section, this member gives the 
  size in bytes of each entry. The member contains 0 if the 
  section does not hold a table of fixed-size entries.

  Flags describes attributes of a section. Flags together with a 
  type defines the purpose of a section. Two sections can be of 
  the same type, but serve different purposes. For example, even 
  though .data and .text share the same type, .data holds the 
  initialized data of a program while .text holds executable 
  instructions of a program. For that reason, .data is given read 
  and write permission, but not executable. Any attempt to 
  execute code in .data is denied by the running OS: in Linux, 
  such invalid section usage gives a segmentation fault.

  ELF gives information to enable an OS with such protection 
  mechanism. However, running on bare metal, nothing can prevent 
  from doing anything. Our OS can execute code in data section, 
  and vice versa, writing to code section.


                                                                                                                                                                                           [Table 5:
Section Flags
]                                                                                                                                                                                           
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Flag  | Descriptions                                                                                                                                                                                                                                                                                                                                                                                        |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| W     | Bytes in this section are writable during execution.                                                                                                                                                                                                                                                                                                                                                |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| A     | Memory is allocated for this section during process execution. 
Some control sections do not reside in the memory image of an 
object file; this attribute is off for those sections.                                                                                                                                                                                                               |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| X     | The section contains executable instructions.                                                                                                                                                                                                                                                                                                                                                       |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| M     | The data in the section may be merged to eliminate duplication. 
Each element in the section is compared against other elements in 
sections with the same name, type and flags. Elements that would 
have identical values at program run-time may be merged.                                                                                                                                      |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| S     | The data elements in the section consist of null-terminated 
character strings. The size of each character is specified in the 
section header's EntSize field.                                                                                                                                                                                                                                     |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| l     | Specific large section for x86_64 architecture. This flag is not 
specified in the Generic ABI but in x86_64 ABI.                                                                                                                                                                                                                                                                                   |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| I     | The Info field of this section header holds an index of a section 
header. Otherwise, the number is the index of something else.                                                                                                                                                                                                                                                                    |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| L     | Preserve section ordering when linking. If this section is 
combined with other sections in the output file, it must appear 
in the same relative order with respect to those sections, as the 
linked-to section appears with respect to sections the linked-to 
section is combined with. Apply when the Link field of this 
section's header references another section (the linked-to 
section) |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| G     | This section is a member (perhaps the only one) of a section 
group.                                                                                                                                                                                                                                                                                                                                |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| T     | This section holds Thread-Local Storage, meaning that each thread 
has its own distinct instance of this data. A thread is a 
distinct execution flow of code. A program can have multiple 
threads that pack different pieces of code and execute 
separately, at the same time. We will learn more about threads 
when writing our kernel.                                                        |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| E     | Link editor is to exclude this section from executable and shared 
library that it builds when those objects are not to be further 
relocated.                                                                                                                                                                                                                                                      |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| x     | Unknown flag to readelf. It happens because the linking process 
can be done manually with a linker like GNU ld (we will later 
later). That is, section flags can be specified manually, and 
some flags are for a customized ELF that the open-source readelf 
doesn't know of.                                                                                                                   |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| O     | This section requires special OS-specific processing (beyond the 
standard linking rules) to avoid incorrect behavior. A link 
editor encounters sections whose headers contain OS-specific 
values it does not recognize by Type or Flags values defined by 
ELF standard, the link editor should combine those sections.                                                                          |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| o     | All bits included in this flag are reserved for operating 
system-specific semantics.                                                                                                                                                                                                                                                                                                               |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| p     | All bits included in this flag are reserved for 
processor-specific semantics. If meanings are specified, the 
processor supplement explains them.                                                                                                                                                                                                                                                  |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


  Link and Info are numbers that references the indexes of 
  sections, symbol table entries, hash table entries. Link field 
  holds the index of a section, while Info field holds an index 
  of a section, a symbol table entry or a hash table entry, 
  depends on the type of a section. 

  Later when writing our OS, we will handcraft the kernel image 
  by explicitly linking the object files (produced by gcc) 
  through a linker script. We will specify the memory layout of 
  sections by specifying at what addresses they will appear in 
  the final image. But we will not assign any section flag and 
  let the linker take care of it. Nevertheless, knowing which 
  flag does what is useful.

  Align is a value that enforces the offset of a section should 
  be divisible by the value. Only 0 and positive integral powers 
  of two are allowed. Values 0 and 1 mean the section has no 
  alignment constraint.

Output of .interp section:

  

  [Nr] Name              Type             Address           
  Offset

       Size              EntSize          Flags  Link  Info  
  Align

  [ 1] .interp           PROGBITS         0000000000400238  
  00000238

       000000000000001c  0000000000000000   A       0     0     1

  

  Nr is 1.

  Type is PROGBITS, which means this section is part of the 
  program.

  Address is 0x0000000000400238, which means the program is 
  loaded at this virtual memory address at runtime.

  Offset is 0x00000238 bytes into file.

  Size is 0x000000000000001c in bytes.

  EntSize is 0, which means this section does not have any 
  fixed-size entry.

  Flags are A (Allocatable), which means this section consumes 
  memory at runtime.

  Info and Link are 0 and 0, which means this section links to no 
  section or entry in any table.

  Align is 1, which means no alignment.



Output of the .text section:

  

  [14] .text             PROGBITS         00000000004003e0  
  000003e0

         0000000000000192  0000000000000000  AX       0     0     
  16

  

  Nr is 14.

  Type is PROGBITS, which means this section is part of the 
  program.

  Address is 0x00000000004003e0, which means the program is 
  loaded at this virtual memory address at runtime.

  Offset is 0x000003e0 bytes into file.

  Size is 0x0000000000000192 in bytes.

  EntSize is 0, which means this section does not have any 
  fixed-size entry.

  Flags are A (Allocatable) and X (Executable), which means this 
  section consumes memory and can be executed as code at runtime.

  Info and Link are 0 and 0, which means this section links to no 
  section or entry in any table.

  Align is 16, which means the starting address of the section 
  should be divisible by 16, or 0x10. Indeed, it is: \mathtt{0x3e0/0x10=0x3e}

  . 

  Understand Section in-depth

In this section, we will learn different details of section types 
and the purposes of special sections e.g. .bss, .text, .data... 
by looking at each section one by one. We will also examine the 
content of each section as a hexdump with the commands:



$ readelf -x <section name|section number> <file>



For example, if you want to examine the content of section with 
index 25 (the .bss section in the sample output) in the file 
hello:



$ readelf -x 25 hello



Equivalently, using name instead of index works:



$ readelf -x .data hello



If a section contains strings e.g. string symbol table, the flag 
-x can be replaced with -p.

  NULL marks a section header as inactive and does not have an 
  associated section. NULL section is always the first entry of 
  section header table. It means, any useful section starts from 
  1.

  The sample output of NULL section:

    

    [Nr] Name             Type             Address           
    Offset

         Size             EntSize          Flags  Link  Info  
    Align

    [ 0]                  NULL             0000000000000000 
    00000000

         0000000000000000 0000000000000000           0     0     
    0

    

  Examining the content, the section is empty:

  

   Section '' has no data to dump.

  

  NOTE marks a section with special information that other 
  programs will check for conformance, compatibility... by a 
  vendor or a system builder.

  In the sample output, we have 2 NOTE sections:

    

    [Nr] Name              Type             Address           
    Offset

         Size              EntSize          Flags  Link  Info  
    Align

    [ 2] .note.ABI-tag     NOTE             0000000000400254  
    00000254

         0000000000000020  0000000000000000   A       0     0     
    4

    [ 3] .note.gnu.build-i NOTE             0000000000400274  
    00000274          

         0000000000000024  0000000000000000   A       0     0     
    4

    

  Examine 2nd section with the command:

  

  $ readelf -x 2 hello

  

  we have:

  

  Hex dump of section '.note.ABI-tag':

    0x00400254 04000000 10000000 01000000 474e5500 
  ............GNU.

    0x00400264 00000000 02000000 06000000 20000000 ............ 
  ...

  

  PROGBITS indicates a section holding the main content of a 
  program, either code or data.

  There are many PROGBITS sections:

    

      [Nr] Name              Type             Address           
    Offset

           Size              EntSize          Flags  Link  Info  
    Align

      [ 1] .interp           PROGBITS         0000000000400238  
    00000238

           000000000000001c  0000000000000000   A       0     0   
      1

      ...

      [11] .init             PROGBITS         0000000000400390  
    00000390

           000000000000001a  0000000000000000  AX       0     0   
      4

      [12] .plt              PROGBITS         00000000004003b0  
    000003b0

           0000000000000020  0000000000000010  AX       0     0   
      16

      [13] .plt.got          PROGBITS         00000000004003d0  
    000003d0

           0000000000000008  0000000000000000  AX       0     0   
      8

      [14] .text             PROGBITS         00000000004003e0  
    000003e0

           0000000000000192  0000000000000000  AX       0     0   
      16

      [15] .fini             PROGBITS         0000000000400574  
    00000574

           0000000000000009  0000000000000000  AX       0     0   
      4

      [16] .rodata           PROGBITS         0000000000400580  
    00000580

           0000000000000004  0000000000000004  AM       0     0   
      4

      [17] .eh_frame_hdr     PROGBITS         0000000000400584  
    00000584

           000000000000003c  0000000000000000   A       0     0   
      4

      [18] .eh_frame         PROGBITS         00000000004005c0  
    000005c0

           0000000000000114  0000000000000000   A       0     0   
      8

      ...

      [23] .got              PROGBITS         0000000000600ff8  
    00000ff8

           0000000000000008  0000000000000008  WA       0     0   
      8

      [24] .got.plt          PROGBITS         0000000000601000  
    00001000

           0000000000000020  0000000000000008  WA       0     0   
      8

      [25] .data             PROGBITS         0000000000601020  
    00001020

           0000000000000010  0000000000000000  WA       0     0   
      8

      [27] .comment          PROGBITS         0000000000000000  
    00001030

           0000000000000034  0000000000000001  MS       0     0   
      1

    

  For our operating system, we only need the following section:

  .text

    This section holds all the compiled code of a program. 

  .data

    This section holds the initialized data of a program. Since 
    the data are initialized with actual values, gcc allocates 
    the section with actual byte in the executable binary.

  .rodata

    This section holds read-only data, such as fixed-size strings 
    in a program, e.g. “Hello World”, and others.

  .bss

    This section, shorts for Block Started by Symbol, holds 
    uninitialized data of a program. Unlike other sections, no 
    space is allocated for this section in the image of the 
    executable binary on disk. The section is allocated only when 
    the program is loaded into main memory.

  Other sections are mainly needed for dynamic linking, that is 
  code linking at runtime for sharing between many programs. To 
  enable such feature, an OS as a runtime environment must be 
  presented. Since we run our OS on bare metal, we are 
  effectively creating such environment. For simplicity, we won't 
  add dynamic linking to our OS.

  SYMTAB and DYNSYM These sections hold symbol table. A symbol 
  table is an array of entries that describe symbols in a 
  program. A symbol is a name assigned to an entity in a program. 
  The types of these entities are also the types of symbols, and 
  these are the possible types of an entity:

  In the sample output, section 5 and 29 are symbol tables:

    

    [Nr] Name              Type             Address           
    Offset

         Size              EntSize          Flags  Link  Info  
    Align

    [ 5] .dynsym           DYNSYM           00000000004002b8  
    000002b8

         0000000000000048  0000000000000018   A       6     1     
    8

    ...

    [29] .symtab           SYMTAB           0000000000000000  
    00001068

         0000000000000648  0000000000000018          30    47     
    8

    

    To show the symbol table:

    

    $ readelf -s hello

    

    Output consists of 2 symbol tables, corresponding to the two 
    sections above, .dynsym and .symtab:

    

    Symbol table '.dynsym' contains 4 entries:

       Num:    Value          Size Type    Bind   Vis      Ndx 
    Name

         0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 

         1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND 
    puts@GLIBC_2.2.5 (2)

         2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND 
    __libc_start_main@GLIBC_2.2.5 (2)

         3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND 
    __gmon_start__

    Symbol table '.symtab' contains 67 entries:

       Num:    Value          Size Type    Bind   Vis      Ndx 
    Name

        ..........................................

        59: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   26 
    _end

        60: 0000000000400430    42 FUNC    GLOBAL DEFAULT   14 
    _start

        61: 0000000000601038     0 NOTYPE  GLOBAL DEFAULT   26 
    __bss_start

        62: 0000000000400526    32 FUNC    GLOBAL DEFAULT   14 
    main

        63: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND 
    _Jv_RegisterClasses

        64: 0000000000601038     0 OBJECT  GLOBAL HIDDEN    25 
    __TMC_END__

        65: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND 
    _ITM_registerTMCloneTable

        66: 00000000004003c8     0 FUNC    GLOBAL DEFAULT   11 
    _init

    

  TLS	The symbol is associated with a Thread-Local Storage 
    entity.

  Num is the index of an entry in a table.

  Value is the virtual memory address where the symbol is 
    located.

  Size is the size of the entity associated with a symbol.

  Type is a symbol type according to table.

    NOTYPE The type of a symbol is not specified. 

    OBJECT	The symbol is associated with a data object. In C, any 
      variable definition is of OBJECT type.

    FUNC The symbol is associated with a function or other 
      executable code. 

    SECTION	The symbol is associated with a section, and exists 
      primarily for relocation.

    FILE The symbol is the name of a source file associated with 
      an executable binary.

    COMMON	The symbol labels an uninitialized variable. That is, 
      when a variable in C is defined as global variable without 
      an initial value, or as an external variable using the 
      extern keyword. In other words, these variables stay in 
      .bss section.

  Bind is the scope of a symbol. 

    LOCAL are symbols that are only visible in the object files 
      that defined them. In C, the static modifier marks a symbol 
      (e.g. a variable/function) as local to only the file that 
      defines it.

      If we define variables and functions with static modifer:

        static int global_static_var = 0;



static void local_func() {

}



int main(int argc, char *argv[])

{

    static int local_static_var = 0;



    return 0;

}

      Then we get the static variables listed as local symbols 
      after compiling:

        

        $ gcc -m32 hello.c -o hello

        $ readelf -s hello

        

        

        Symbol table '.dynsym' contains 5 entries:

           Num:    Value  Size Type    Bind   Vis      Ndx Name

             0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 

             1: 00000000     0 FUNC    GLOBAL DEFAULT  UND 
        puts@GLIBC_2.0 (2)

             2: 00000000     0 NOTYPE  WEAK   DEFAULT  UND 
        __gmon_start__

             3: 00000000     0 FUNC    GLOBAL DEFAULT  UND 
        __libc_start_main@GLIBC_2.0 (2)

             4: 080484bc     4 OBJECT  GLOBAL DEFAULT   16 
        _IO_stdin_used

        Symbol table '.symtab' contains 72 entries:

           Num:    Value  Size Type    Bind   Vis      Ndx Name

             0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 

                ......... output omitted .........

            38: 0804a020     4 OBJECT  LOCAL  DEFAULT   26 
        global_static_var

            39: 0804840b     6 FUNC    LOCAL  DEFAULT   14 
        local_func

            40: 0804a024     4 OBJECT  LOCAL  DEFAULT   26 
        local_static_var.1938

         ......... output omitted .........

        

    GLOBAL	are symbols that are accessible by other object files 
      when linking together. These symbols are primarily 
      non-static functions and non-static global data. The extern 
      modifier marks a symbol as externally defined elsewhere but 
      is accessible in the final executable binary, so an extern 
      variable is also considered GLOBAL.

      Similar to the LOCAL example above, the output lists many 
      GLOBAL symbols such as main:

        Num:    Value  Size Type    Bind   Vis      Ndx Name

        ......... output omitted .........

         66: 080483e1    10 FUNC    GLOBAL DEFAULT   14 main

        ......... output omitted .........

    WEAK are symbols whose definitions can be redefined. 
      Normally, a symbol with multiple definitions are reported 
      as an error by a compiler. However, this constraint is lax 
      when a definition is explicitly marked as weak, which means 
      the default implementation can be replaced by a different 
      definition at link time.

      Suppose we have a default implementation of the function 
      add:

        #include <stdio.h>



__attribute__((weak)) int add(int a, int b) {

    printf("warning: function is not implemented.\n");

    return 0;

}



int main(int argc, char *argv[])

{

    printf("add(1,2) is %d\n", add(1,2));

    return 0;

}

        __attribute__((weak)) is a [margin:
function attribute
]function attribute. A function attributefunction attribute is 
        extra information for a compiler to handle a function 
        differently from a normal function. In this example, weak 
        attribute makes the function add a weak function,which 
        means the default implementation can be replaced by a 
        different definition at link time. Function attribute is 
        a feature of a compiler, not standard C.

        If we do not supply a different function definition in a 
        different file (must be in a different file, otherwise 
        gcc reports as an error), then the default implementation 
        is applied. When the function add is called, it only 
        prints the message: "warning: function not 
        implemented"and returns 0:

        

        $ ./hello 

        warning: function is not implemented.

        add(1,2) is 0

        

        However, if we supply a different definition in another 
        file e.g. math.c:

        int add(int a, int b) {

    return a + b;

}

        and compile the two files together:

        

        $ gcc math.c hello.c -o hello

        

        Then, when running hello, no warning message is printed 
        and the correct value is returned.

        Weak symbol is a mechanism to provide a default 
        implementation, but replaceable when a better 
        implementation is available (e.g. more specialized and 
        optimized) at link-time.

  Vis is the visibility of a symbol. The following values are 
    available:

    
                                                                                                                                  [Table 6:
Symbol Visibility
]                                                                                                                                   
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Value      | Description                                                                                                                                                                                                                                                                       |
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | DEFAULT    | The visibility is specified by the binding type of asymbol. 

• Global and weak symbols are visible outside of their defining 
  component (executable file or shared object).

• Local symbols are hidden. See HIDDEN below.                                                     |
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | HIDDEN     | A symbol is hidden when the name is not visible to any other 
program outside of its running program.                                                                                                                                                                             |
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | PROTECTED  | A symbol is protected when it is shared outside of its running 
program or shared libary and cannot be overridden. That is, there 
can only be one definition for this symbol across running 
programs that use it. No program can define its own definition of 
the same symbol. |
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | INTERNAL   | Visibility is processor-specific and is defined by 
processor-specific ABI.                                                                                                                                                                                                       |
    +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    

  Ndx is the index of a section that the symbol is in. Aside from 
    fixed index numbers that represent section indexes, index has 
    these special values:

    
                                                                                                                                                 [Table 7:
Symbol Index
]                                                                                                                                                 
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Value                 | Description                                                                                                                                                                                                                                                                                    |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | ABS                   | The index will not be changed by any symbol relocation.                                                                                                                                                                                                                                        |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | COM                   | The index refers to an unallocated common block.                                                                                                                                                                                                                                               |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | UND                   | The symbol is undefined in the current object file, which means 
the symbol depends on the actual definition in another file. 
Undefined symbols appears when the object file refers to symbols 
that are available at runtime, from shared library.                                           |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | LORESERVE

HIRESERVE  | LORESERVE is the lower boundary of the reserve indexes. Its value 
is 0xff00.

HIREVERSE is the upper boundary of the reserve indexes. Its value 
is 0xffff.

The operating system reserves exclusive indexes between LORESERVE 
and HIRESERVE, which do not map to any actual section header. |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | XINDEX                | The index is larger than LORESERVE. The actual value will be 
contained in the section SYMTAB_SHNDX, where each entry is a 
mapping between a symbol, whose Ndx field is a XINDEX value, and 
the actual index value.                                                                          |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Others                | Sometimes, values such as ANSI_COM, LARGE_COM, SCOM, SUND appear. 
This means that the index is processor-specific.                                                                                                                                                                            |
    +-----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    

  Name is the symbol name.

  A C application program always starts from symbol main. The 
  entry for main in the symbol table in .symtab section is:

    

    Num:                Value  Size Type    Bind   Vis      Ndx 
    Name

     62:     0000000000400526    32 FUNC    GLOBAL DEFAULT   14 
    main

    

  The entry shows that:

  • main is the 62[superscript:th] entry in the table.

  • main starts at address 0x0000000000400526.

  • main consumes 32 bytes.

  • main is a function.

  • main is in global scope.

  • main is visible to other object files that use it.

  • main is inside the 14[superscript:th] section, which is .text. This is logical, since .text holds all 
    program code.

  STRTAB hold a table of null-terminated strings, called string 
  table. The first and last byte of this section is always a NULL 
  character. A string table section exists because a string can 
  be reused by more than one section to represent symbol and 
  section names, so a program like readelf or objdump can display 
  various objects in a program, e.g. variable, functions, section 
  names, in a human-readable text instead of its raw hex address.

  In the sample output, section 28 and 30 are of STRTAB type:

    

    [Nr] Name              Type             Address           
    Offset

         Size              EntSize          Flags  Link  Info  
    Align

    [28] .shstrtab         STRTAB           0000000000000000  
    000018b6

         000000000000010c  0000000000000000           0     0     
    1

    [30] .strtab           STRTAB           0000000000000000  
    000016b0

         0000000000000206  0000000000000000           0     0     
    1

    

  .shstrtab holds all the section names.

  .strtab holds the symbols e.g. variable names, function names, 
    struct names, etc., in a C program, but not fixed-size 
    null-terminated C strings; the C strings are kept in .rodata 
    section.

  Strings in those section can be inspected with the command:

    

    $ readelf -p 29 hello

    

    The output shows all the section names, with the offset (also 
    the string index) into .shstrtab the table to the left:

    

    String dump of section '.shstrtab':  

      [     1]  .symtab

      [     9]  .strtab

      [    11]  .shstrtab

      [    1b]  .interp

      [    23]  .note.ABI-tag

      [    31]  .note.gnu.build-id

      [    44]  .gnu.hash

      [    4e]  .dynsym

      [    56]  .dynstr

      [    5e]  .gnu.version

      [    6b]  .gnu.version_r

      [    7a]  .rela.dyn

      [    84]  .rela.plt

      [    8e]  .init

      [    94]  .plt.got

      [    9d]  .text

      [    a3]  .fini

      [    a9]  .rodata

      [    b1]  .eh_frame_hdr

      [    bf]  .eh_frame

      [    c9]  .init_array

      [    d5]  .fini_array

      [    e1]  .jcr

      [    e6]  .dynamic

      [    ef]  .got.plt

      [    f8]  .data

      [    fe]  .bss

      [   103]  .comment

    

    The actual implementation of a string table is a contiguous 
    array of null-terminated strings. The index of a string is 
    the position of its first character in the array. For 
    example, in the above string table, .symtab is at index 1 in 
    the array (NULL character is at index 0). The length of 
    .symtab is 7, plus the NULL character, which occurs 8 bytes 
    in total. So, .strtab starts at index 9, and so on.

    

    
            +-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+----+
                | 00  | 01  | 02  | 03  | 04  | 05  | 06  | 07  | 08  | 09  | 0a  | 0b  | 0c  | 0d  | 0e  | 0f |
    +-----------+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+----+
    | 00000000  | \0  | .   | s   | y   | m   | t   | a   | b   | \0  | .   | s   | t   | r   | t   | a   | b  |
    +-----------+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+----+
                                                                                                                
                +-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+----+
                | 00  | 01  | 02  | 03  | 04  | 05  | 06  | 07  | 08  | 09  | 0a  | 0b  | 0c  | 0d  | 0e  | 0f |
    +-----------+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+----+
    | 00000010  | \0  | .   | s   | h   | s   | t   | r   | t   | a   | b   | \0  | .   | i   | n   | t   | e  |
    +-----------+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+----+
                                                .... and so on ....                                             
    

    

    



    Similarly, the output of .strtab:

    

    String dump of section '.strtab':

      [     1]  crtstuff.c

      [     c]  __JCR_LIST__

      [    19]  deregister_tm_clones

      [    2e]  __do_global_dtors_aux

      [    44]  completed.7585

      [    53]  __do_global_dtors_aux_fini_array_entry

      [    7a]  frame_dummy

      [    86]  __frame_dummy_init_array_entry

      [    a5]  hello.c

      [    ad]  __FRAME_END__

      [    bb]  __JCR_END__

      [    c7]  __init_array_end

      [    d8]  _DYNAMIC

      [    e1]  __init_array_start

      [    f4]  __GNU_EH_FRAME_HDR

      [   107]  _GLOBAL_OFFSET_TABLE_

      [   11d]  __libc_csu_fini

      [   12d]  _ITM_deregisterTMCloneTable

      [   149]  j

      [   14b]  _edata

      [   152]  __libc_start_main@@GLIBC_2.2.5

      [   171]  __data_start

      [   17e]  __gmon_start__

      [   18d]  __dso_handle

      [   19a]  _IO_stdin_used

      [   1a9]  __libc_csu_init

      [   1b9]  __bss_start

      [   1c5]  main

      [   1ca]  _Jv_RegisterClasses

      [   1de]  __TMC_END__

      [   1ea]  _ITM_registerTMCloneTable

    

  HASH holds a symbol hash table, which supports symbol table 
  access.

  DYNAMIC holds information for dynamic linking. 

  NOBITS is similar to PROGBITS but occupies no space.

  .bss section holds uninitialized data, which means the bytes in 
  the section can have any value. Until a operating system 
  actually loads the section into main memory, there is no need 
  to allocate space for the binary image on disk to reduce the 
  size of a binary file. Here is the details of .bss from the 
  example output:

  

  [Nr] Name              Type             Address           
  Offset

       Size              EntSize          Flags  Link  Info  
  Align

  [26] .bss              NOBITS           0000000000601038  
  00001038

       0000000000000008  0000000000000000  WA       0     0     1 
    

  [27] .comment          PROGBITS         0000000000000000  
  00001038

       0000000000000034  0000000000000001  MS       0     0     1 

  

  In the above output, the size of the section is only 8 bytes, 
  while the offsets of both sections are the same, which means 
  .bss consumes no byte of the executable binary on disk. 

  Notice that the .comment section has no starting address. This 
  means that this section is discarded when the executable binary 
  is loaded into memory.

  REL holds relocation entries without explicit addends. This 
  type will be explained in details in [sec:Understand-relocations-with-readelf]

  RELA holds relocation entries with explicit addends. This type 
  will be explained in details in [sec:Understand-relocations-with-readelf]

  INIT_ARRAY is an array of function pointers for program 
  initialization. When an application program runs, before 
  getting to main(), initialization code in .init and this 
  section are executed first. The first element in this array is 
  an ignored function pointer. 

  It might not make sense when we can include initialization code 
  in the main() function. However, for shared object files where 
  there are no main(), this section ensures that the 
  initialization code from an object file executes before any 
  other code to ensure a proper environment for main code to run 
  properly. It also makes an object file more modularity, as the 
  main application code needs not to be responsible for 
  initializing a proper environment for using a particular object 
  file, but the object file itself. Such a clear division makes 
  code cleaner.

  However, we will not use any .init and INIT_ARRAY sections in 
  our operating system, for simplicity, as initializing an 
  environment is part of the operating-system domain.

  To use the INIT_ARRAY, we simply mark a function with the 
  attribute constructor:

    #include <stdio.h>



__attribute__((constructor)) static void init1(){

    printf("%s\n", __FUNCTION__);

}



__attribute__((constructor)) static void init2(){

    printf("%s\n", __FUNCTION__);

}





int main(int argc, char *argv[])

{

    printf("hello world\n");



    return 0;

}

    The program automatically calls the constructor without 
    explicitly invoking it:

    

    $ gcc -m32 hello.c -o hello

    $ ./hello 

    init1

    init2

    hello world

    

  

  Optionally, a constructor can be assigned with a priority from 
  101 onward. The priorities from 0 to 100 are reserved for gcc. 
  If we want init2 to run before init1, we give it a higher 
  priority:

    #include <stdio.h>



__attribute__((constructor(102))) static void init1(){

    printf("%s\n", __FUNCTION__);

}



__attribute__((constructor(101))) static void init2(){

    printf("%s\n", __FUNCTION__);

}





int main(int argc, char *argv[])

{

    printf("hello world\n");



    return 0;

}

    The call order should be exactly as specified:

  

  $ gcc -m32 hello.c -o hello

  $ ./hello

  init2

  init1

  hello world

  

  

  We can add initialization functions using another method:

    #include <stdio.h>



void init1() {

    printf("%s\n", __FUNCTION__);

}



void init2() {

    printf("%s\n", __FUNCTION__);

}



/* Without typedef, init is a definition of a function pointer.

   With typedef, init is a declaration of a type.*/

typedef void (*init)();



__attribute__((section(".init_array"))) init init_arr[2] = 
{init1, init2};



int main(int argc, char *argv[])

{

    printf("hello world!\n");



    return 0;

}

    The attribute section(“...”) put a function into a particular 
    section rather then the default .text. In this example, it is 
    .init_arary. Again, the program automatically calls the 
    constructors without explicitly invoking it:

    

    $ gcc -m32 hello.c -o hello

    $ ./hello 

    init1

  init2

  hello world!

    

  FINI_ARRAY is an array of function pointers for program 
  termination, called after exiting main(). If the application 
  terminate abnormally, such as through abort() call or a crash, 
  the .finit_array is ignored.

  A destructor is automatically called after exiting main(), if 
  one or more available:

  #include <stdio.h>



__attribute__((destructor)) static void destructor(){

    printf("%s\n", __FUNCTION__);

}



int main(int argc, char *argv[])

{

    printf("hello world\n");



    return 0;

}

  

  $ gcc -m32 hello.c -o hello

  $ ./hello 

  hello world

  destructor

  

  PREINIT_ARRAY is an array of function pointers that are invoked 
  before all other initialization functions in INIT_ARRAY.

  To use the .preinit_array, the only way to put functions into 
  this section is to use the attribute section():

  #include <stdio.h>



void preinit1() {

    printf("%s\n", __FUNCTION__);

}



void preinit2() {

    printf("%s\n", __FUNCTION__);

}



void init1() {

    printf("%s\n", __FUNCTION__);

}



void init2() {

    printf("%s\n", __FUNCTION__);

}





typedef void (*preinit)();

typedef void (*init)();



__attribute__((section(".preinit_array"))) preinit preinit_arr[2] = 
{preinit1, preinit2};

__attribute__((section(".init_array"))) init init_arr[2] = 
{init1, init2};



int main(int argc, char *argv[])

{

    printf("hello world!\n");



    return 0;

}

  

  $ gcc -m32 hello2.c -o hello2

  $ ./hello2

  preinit1

  preinit2

  init1

  init2

  hello world!

  

  GROUP defines a section group, which is the same section that 
  appears in different object files but when merged into the 
  final executable binary file, only one copy is kept and the 
  rest in other object files are discarded. This section is only 
  relevant in C++ object files, so we will not examine further.

  SYMTAB_SHNDX is a section containing extended section indexes, 
  that are associated with a symbol table. This section only 
  appears when the Ndx value of an entry in the symbol table 
  exceeds the LORESERVE value. This section then maps between a 
  symbol and an actual index value of a section header.

Upon understanding section types, we can understand the number in 
Link and Info fields:




+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Type            | Link                                                                           | Info                                                                                                                                                    |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| DYNAMIC         | Entries in this section uses the section index of the dynamic 
string table.   | 0                                                                                                                                                       |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| HASH

GNU_HASH  | The section index of the symbol table to which the hash table 
applies.        | 0                                                                                                                                                       |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| REL

RELA       | The section index of the associated symbol table.                              | The section index to which the relocation applies.                                                                                                      |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| SYMTAB

DYNSYM  | The section index of the associated string table.                              | One greater than the symbol table index of the last local symbol.                                                                                       |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| GROUP           | The section index of the associated symbol table.                              | The symbol index of an entry in the associated symbol table. The 
name of the specified symbol table entry provides a signature for 
the section group. |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| SYMTAB_SHNDX    | The section header index of the associated symbol table.                       |                                                                                                                                                         |
+-----------------+--------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+




Verify that the value of the Link field of a SYMTAB section is 
the index of a STRTAB section.



Verify that the value of the Info field of a SYMTAB section is 
the index of last local symbol + 1. It means, in the symbol 
table, from the index listed by Info field onward, no local 
symbol appears.



Verify that the value of the Info field of a REL section is the 
index of the SYMTAB section.



Verify that the value of the Link field of a REL section is the 
index of the section where relocation is applied. For example. if 
the section is .rel.text, then the relocating section should be 
.text.

  Program header table<sec:Program-header-table>

A program header tableprogram header table is an array of program 
headers that defines the memory layout of a program at runtime. 

A program headerprogram header is a description of a program 
segment.

A program segmentprogram segment is a collection of related 
sections. A segment contains zero or more sections. An operating 
system when loading a program, only use segments, not sections. 
To see the information of a program header table, we use the -l 
option with readelf:



$ readelf -l <binary file>



Similar to a section, a program header also has types:

  PHDR specifies the location and size of the program header 
  table itself, both in the file and in the memory image of the 
  program

  INTERP specifies the location and size of a null-terminated 
  path name to invoke as an interpreter for linking runtime 
  libraries.

  LOAD specifies a loadable segment. That is, this segment is 
  loaded into main memory.

  DYNAMIC specifies dynamic linking information.

  NOTE specifies the location and size of auxiliary information.

  TLS specifies the Thread-Local Storage template, which is 
  formed from the combination of all sections with the flag TLS.

  GNU_STACK indicates whether the program's stack should be made 
  executable or not. Linux kernel uses this type.

A segment also has permission, which is a combination of these 3 
values:[float MarginTable:
[MarginTable 3:
Segment Permission
]


+-------------+-------------+
| Permission  | Description |
+-------------+-------------+
+-------------+-------------+
| R           | Readable    |
+-------------+-------------+
| W           | Writable    |
+-------------+-------------+
| E           | Executable  |
+-------------+-------------+

]

• Read (R)

• Write (W)

• Execute (E)


-------------------------------------------


The command to get the program header table:

  

  $ readelf -l hello

  

  Output:

  

  Elf file type is EXEC (Executable file)

  Entry point 0x400430

  There are 9 program headers, starting at offset 64

  

  Program Headers:

    Type           Offset             VirtAddr           PhysAddr

                   FileSiz            MemSiz              Flags  
  Align

    PHDR           0x0000000000000040 0x0000000000400040 
  0x0000000000400040

                   0x00000000000001f8 0x00000000000001f8  R E    
  8

    INTERP         0x0000000000000238 0x0000000000400238 
  0x0000000000400238

                   0x000000000000001c 0x000000000000001c  R      
  1

        [Requesting program interpreter: 
  /lib64/ld-linux-x86-64.so.2]

    LOAD           0x0000000000000000 0x0000000000400000 
  0x0000000000400000

                   0x000000000000070c 0x000000000000070c  R E    
  200000

    LOAD           0x0000000000000e10 0x0000000000600e10 
  0x0000000000600e10

                   0x0000000000000228 0x0000000000000230  RW     
  200000

    DYNAMIC        0x0000000000000e28 0x0000000000600e28 
  0x0000000000600e28

                   0x00000000000001d0 0x00000000000001d0  RW     
  8

    NOTE           0x0000000000000254 0x0000000000400254 
  0x0000000000400254

                   0x0000000000000044 0x0000000000000044  R      
  4

    GNU_EH_FRAME   0x00000000000005e4 0x00000000004005e4 
  0x00000000004005e4

                   0x0000000000000034 0x0000000000000034  R      
  4

    GNU_STACK      0x0000000000000000 0x0000000000000000 
  0x0000000000000000

                   0x0000000000000000 0x0000000000000000  RW     
  10

    GNU_RELRO      0x0000000000000e10 0x0000000000600e10 
  0x0000000000600e10

                   0x00000000000001f0 0x00000000000001f0  R      
  1

  

   Section to Segment mapping:

    Segment Sections...

     00     

     01     .interp

     02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash 
  .dynsym .dynstr 

  .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt 
  .plt.got .text .fini

  .rodata .eh_frame_hdr .eh_frame 

     03     .init_array .fini_array .jcr .dynamic .got .got.plt 
  .data .bss 

     04     .dynamic 

     05     .note.ABI-tag .note.gnu.build-id 

     06     .eh_frame_hdr 

     07     

     08     .init_array .fini_array .jcr .dynamic .got 

  

  In the sample output, LOAD segment appears twice:

  

  LOAD           0x0000000000000000 0x0000000000400000 
  0x0000000000400000

                 0x000000000000070c 0x000000000000070c  R E    
  200000

  LOAD           0x0000000000000e10 0x0000000000600e10 
  0x0000000000600e10

                 0x0000000000000228 0x0000000000000230  RW     
  200000

  

  Why? Notice the permission: 

  • the upper LOAD has Read and Execute permission. This is a 
    text segment. A text segment contains read-only instructions 
    and read-only data.

  • the lower LOAD has Read and Write permission. This is a data 
    segment. It means that this segment can be read and written 
    to, but is not allowed to be used as executable code, for 
    security reason.

  Then, LOAD contains the following sections:

  

     02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash 
  .dynsym .dynstr 

  .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt 
  .plt.got .text .fini 

  .rodata .eh_frame_hdr .eh_frame 

     03     .init_array .fini_array .jcr .dynamic .got .got.plt 
  .data .bss 

  

  The first number is the index of a program header in program 
  header table, and the remaining text is the list of all 
  sections within a segment. Unfortunately, readelf does not 
  print the index, so a user needs to keep track manually which 
  segment is of which index. First segment starts at index 0, 
  second at index 1 and so on. LOAD are segments at index 2 and 
  1. As can be seen from the two lists of sections, most sections 
  are loadable and is available at runtime.


-------------------------------------------


  Segments vs sections

As mentioned earlier, an operating system loads program segments, 
not sections. However, a question arises: Why doesn't the 
operating system use sections instead? After all, a section also 
contains similar information to a program segment, such as the 
type, the virtual memory address to be loaded, the size, the 
attributes, the flags and align. As explained before, a segment 
is the perspective of an operating system, while a section is the 
perspective of a linker. To understand why, looking into the 
structure of a segment, we can easily see:

• A segment is a collection of sections. It means that sections 
  are logically grouped together by their attributes. For 
  example, all sections in a LOAD segment are always loaded by 
  the operating system; all sections have the same permission, 
  either a RE (Read + Execute) for executable sections, or RW 
  (Read + Write) for data sections.

• By grouping sections into a segment, it is easier for an 
  operating system to batch load sections just once by loading 
  the start and end of a segment, instead of loading section by 
  section.

• Since a segment is for loading a program and a section is for 
  linking a program, all the sections in a segment is within its 
  start and end virtual memory addresses of a segment.

To see the last point clearer, consider an example of linking two 
object files. Suppose we have two source files:

#include <stdio.h>



int main(int argc, char *argv[])

{

    printf("Hello World\n");

    return 0;

}

and:

int add(int a, int b) {

    return a + b;

}

Now, compile the two source files as object files:



$ gcc -m32 -c math.c 

$ gcc -m32 -c hello.c



Then, we check the sections of math.o:



$ readelf -S math.o





There are 11 section headers, starting at offset 0x1a8:

Section Headers:

  [Nr] Name              Type            Addr     Off    Size   
ES Flg Lk Inf Al

  [ 0]                   NULL            00000000 000000 000000 
00      0   0  0

  [ 1] .text             PROGBITS        00000000 000034 00000d 
00  AX  0   0  1

  [ 2] .data             PROGBITS        00000000 000041 000000 
00  WA  0   0  1

  [ 3] .bss              NOBITS          00000000 000041 000000 
00  WA  0   0  1

  [ 4] .comment          PROGBITS        00000000 000041 000035 
01  MS  0   0  1

  [ 5] .note.GNU-stack   PROGBITS        00000000 000076 000000 
00      0   0  1

  [ 6] .eh_frame         PROGBITS        00000000 000078 000038 
00   A  0   0  4

  [ 7] .rel.eh_frame     REL             00000000 00014c 000008 
08   I  9   6  4

  [ 8] .shstrtab         STRTAB          00000000 000154 000053 
00      0   0  1

  [ 9] .symtab           SYMTAB          00000000 0000b0 000090 
10     10   8  4

  [10] .strtab           STRTAB          00000000 000140 00000c 
00      0   0  1

Key to Flags:

  W (write), A (alloc), X (execute), M (merge), S (strings)

  I (info), L (link order), G (group), T (TLS), E (exclude), x 
(unknown)

  O (extra OS processing required) o (OS specific), p (processor 
specific)



As shown in the output, all the section virtual memory addresses 
of every section are set to 0. At this stage, each object file is 
simply a block of binary that contains code and data. Its 
existence is to serve as a material container for the final 
product, which is the executable binary. As such, the virtual 
addresses in hello.o are all zeroes.

No segment exists at this stage:



$ readelf -l math.o

There are no program headers in this file.



The same happens to other object file:



There are 13 section headers, starting at offset 0x224:

Section Headers:

  [Nr] Name              Type            Addr     Off    Size   
ES Flg Lk Inf Al

  [ 0]                   NULL            00000000 000000 000000 
00      0   0  0

  [ 1] .text             PROGBITS        00000000 000034 00002e 
00  AX  0   0  1

  [ 2] .rel.text         REL             00000000 0001ac 000010 
08   I 11   1  4

  [ 3] .data             PROGBITS        00000000 000062 000000 
00  WA  0   0  1

  [ 4] .bss              NOBITS          00000000 000062 000000 
00  WA  0   0  1

  [ 5] .rodata           PROGBITS        00000000 000062 00000c 
00   A  0   0  1

  [ 6] .comment          PROGBITS        00000000 00006e 000035 
01  MS  0   0  1

  [ 7] .note.GNU-stack   PROGBITS        00000000 0000a3 000000 
00      0   0  1

  [ 8] .eh_frame         PROGBITS        00000000 0000a4 000044 
00   A  0   0  4

  [ 9] .rel.eh_frame     REL             00000000 0001bc 000008 
08   I 11   8  4

  [10] .shstrtab         STRTAB          00000000 0001c4 00005f 
00      0   0  1

  [11] .symtab           SYMTAB          00000000 0000e8 0000b0 
10     12   9  4

  [12] .strtab           STRTAB          00000000 000198 000013 
00      0   0  1

Key to Flags:

  W (write), A (alloc), X (execute), M (merge), S (strings)

  I (info), L (link order), G (group), T (TLS), E (exclude), x 
(unknown)

  O (extra OS processing required) o (OS specific), p (processor 
specific)





$ readelf -l hello.o

There are no program headers in this file.



Only when object files are combined into a final executable 
binary, sections are fully realized:



$ gcc -m32 math.o hello.o -o hello

$ readelf -S hello.





There are 31 section headers, starting at offset 0x1804:

Section Headers:

  [Nr] Name              Type            Addr     Off    Size   
ES Flg Lk Inf Al

  [ 0]                   NULL            00000000 000000 000000 
00      0   0  0

  [ 1] .interp           PROGBITS        08048154 000154 000013 
00   A  0   0  1

  [ 2] .note.ABI-tag     NOTE            08048168 000168 000020 
00   A  0   0  4

  [ 3] .note.gnu.build-i NOTE            08048188 000188 000024 
00   A  0   0  4

  [ 4] .gnu.hash         GNU_HASH        080481ac 0001ac 000020 
04   A  5   0  4

  [ 5] .dynsym           DYNSYM          080481cc 0001cc 000050 
10   A  6   1  4

  [ 6] .dynstr           STRTAB          0804821c 00021c 00004a 
00   A  0   0  1

  [ 7] .gnu.version      VERSYM          08048266 000266 00000a 
02   A  5   0  2

  [ 8] .gnu.version_r    VERNEED         08048270 000270 000020 
00   A  6   1  4

  [ 9] .rel.dyn          REL             08048290 000290 000008 
08   A  5   0  4

  [10] .rel.plt          REL             08048298 000298 000010 
08  AI  5  24  4

  [11] .init             PROGBITS        080482a8 0002a8 000023 
00  AX  0   0  4

  [12] .plt              PROGBITS        080482d0 0002d0 000030 
04  AX  0   0 16

  [13] .plt.got          PROGBITS        08048300 000300 000008 
00  AX  0   0  8

  [14] .text             PROGBITS        08048310 000310 0001a2 
00  AX  0   0 16

  [15] .fini             PROGBITS        080484b4 0004b4 000014 
00  AX  0   0  4

  [16] .rodata           PROGBITS        080484c8 0004c8 000014 
00   A  0   0  4

  [17] .eh_frame_hdr     PROGBITS        080484dc 0004dc 000034 
00   A  0   0  4

  [18] .eh_frame         PROGBITS        08048510 000510 0000ec 
00   A  0   0  4

  [19] .init_array       INIT_ARRAY      08049f08 000f08 000004 
00  WA  0   0  4

  [20] .fini_array       FINI_ARRAY      08049f0c 000f0c 000004 
00  WA  0   0  4

  [21] .jcr              PROGBITS        08049f10 000f10 000004 
00  WA  0   0  4

  [22] .dynamic          DYNAMIC         08049f14 000f14 0000e8 
08  WA  6   0  4

  [23] .got              PROGBITS        08049ffc 000ffc 000004 
04  WA  0   0  4

  [24] .got.plt          PROGBITS        0804a000 001000 000014 
04  WA  0   0  4

  [25] .data             PROGBITS        0804a014 001014 000008 
00  WA  0   0  4

  [26] .bss              NOBITS          0804a01c 00101c 000004 
00  WA  0   0  1

  [27] .comment          PROGBITS        00000000 00101c 000034 
01  MS  0   0  1

  [28] .shstrtab         STRTAB          00000000 0016f8 00010a 
00      0   0  1

  [29] .symtab           SYMTAB          00000000 001050 000470 
10     30  48  4

  [30] .strtab           STRTAB          00000000 0014c0 000238 
00      0   0  1

Key to Flags:

  W (write), A (alloc), X (execute), M (merge), S (strings)

  I (info), L (link order), G (group), T (TLS), E (exclude), x 
(unknown)

  O (extra OS processing required) o (OS specific), p (processor 
specific)



Every loadable section is assigned an address, highlighted in 
green. The reason each section got its own address is that in 
reality, gcc does not combine an object by itself, but invokes 
the linker ld. The linker ld uses the default script that it can 
find in the system to build the executable binary. In the default 
script, a segment is assigned a starting address 0x8048000 and 
sections belong to it. Then:

• \mathtt{1^{st}\,section\,address=starting\,segment\,address+section\,offset=0x8048000+0x154=0x08048154}


• \mathtt{2^{nd}\,section\,address=starting\,segment\,address+section\,offset=0x8048000+0x168=0x08048168}


• .... and so on until the last loadable section...

Indeed, the end address of a segment is also the end address of 
the final section. We can see this by listing all the segments:



$ readelf -l hello



And check, for example, LOAD segment which starts at 0x08048000 
and end at \mathtt{0x08048000+0x005fc=0x080485fc}
:



Elf file type is EXEC (Executable file)

Entry point 0x8048310

There are 9 program headers, starting at offset 52

Program Headers:

  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  
Flg Align

  PHDR           0x000034 0x08048034 0x08048034 0x00120 0x00120 R 
E 0x4

  INTERP         0x000154 0x08048154 0x08048154 0x00013 0x00013 R 
  0x1

      [Requesting program interpreter: /lib/ld-linux.so.2]

  LOAD           0x000000 0x08048000 0x08048000 0x005fc 0x005fc R 
E 0x1000

  LOAD           0x000f08 0x08049f08 0x08049f08 0x00114 0x00118 
RW  0x1000

  DYNAMIC        0x000f14 0x08049f14 0x08049f14 0x000e8 0x000e8 
RW  0x4

  NOTE           0x000168 0x08048168 0x08048168 0x00044 0x00044 R 
  0x4

  GNU_EH_FRAME   0x0004dc 0x080484dc 0x080484dc 0x00034 0x00034 R 
  0x4

  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 
RW  0x10

  GNU_RELRO      0x000f08 0x08049f08 0x08049f08 0x000f8 0x000f8 R 
  0x1

 Section to Segment mapping:

  Segment Sections...

   00     

   01     .interp 

   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash 
.dynsym .dynstr 

.gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .plt.got 
.text .fini 

.rodata .eh_frame_hdr .eh_frame 

   03     .init_array .fini_array .jcr .dynamic .got .got.plt 
.data .bss 

   04     .dynamic 

   05     .note.ABI-tag .note.gnu.build-id 

   06     .eh_frame_hdr 

   07     

   08     .init_array .fini_array .jcr .dynamic .got 



The last section in the first LOAD segment is .eh_frame. The 
section starts at 0x08048510, with the offset 0x510 and its size 
is 0xec. The end address of .eh_frame should be: \mathtt{0x08048510+0x510+0xec=0x080485fc}

, exactly the same as the end address of the first LOAD segment.

Chapter [chap:Linking-and-loading] will explore this whole 
process in detail.