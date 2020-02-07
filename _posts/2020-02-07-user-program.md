---
title: 4. 用户应用程序
date: 2019-06-07
---

`xv6` 的第一个应用程序是 `init` 进程，其前身是 `initcode.S` 代码。其他进程可以从磁盘中读入， 例如 `sh`、`wc` 等。需要注意，这些程序不属于内核代码。

## 1. 运行程序

在 shell 执行磁盘上的外部命令时，将通过 `sys_exec` 系统调用（定义于 [sysfile.c#L397](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L397)，被 `xv6` 组织在文件系统部分的 `sysfile.c` 中，可能考虑到从磁盘读入可执行文件影像的原因）而执行内核函数 `exec()`。第一个进程 `init` 开始是内核代码自带的，但它很快就执行磁盘上的 `init` 程序，也是利用 `exec()` 完成的。

需要给 `exec()` 提供可执行文件名以及相应的命令行参数，该函数定义为 `int  exec(char *path, char **argv)` （参见 [exec.c#L10](https://github.com/professordeng/xv6-expansion/blob/master/exec.c#L10)），函数成功返回值为 0，出错返回 `-1`。 

`exec()` 需要建立进程的虚存镜像（即进程页表），包括用户空间和内核空间：`setupkvm()` 创建内核部分页表、由 `allocuvm()` 和 `loaduvm()` 创建用户态的内存镜像。

### 1.1 查找可执行文件

[exec.c#L22](https://github.com/professordeng/xv6-expansion/blob/master/exec.c#L22) 根据传入的文件名调用 `namei()` 查找对应的索引节点 `ip`，然后 [exec.c#L32](https://github.com/professordeng/xv6-expansion/blob/master/exec.c#L32) 检查对应的文件是否符合 ELF 可执行文件格式，即是否包含 `ELF_MAGIC`。

### 1.2 装入可执行文件

[exec.c#L41](https://github.com/professordeng/xv6-expansion/blob/master/exec.c#L41) 根据可执行文件的程序头表，逐项扫描遍历，对于类型为 `ELF_PROG_LOAD` 的段将会分配内存完成装载。 

对于每一个需要装入的段，先 `allocuvm()` 分配足够的空间并建立页表映射，然后通过
`loaduvm()` 从磁盘再读入该段的内容。实际上利用 `readelf -h _ls` 检查 `_ls` 可执行文件，发现它只有一个需要装载的段，这与普通 Linux 可执行文件分成代码和数据两个可装载段的安排不同。

完成可执行文件的段的载入过程后，继续分配两个页，第一个页为不可访问页，第二个页作为用户堆栈。此时的进程空间安排为：

代码 - 数据 - `bss` - 禁止访问页 - 用户堆栈

### 1.3 准备命令行参数

[exec.c#L63](https://github.com/professordeng/xv6-expansion/blob/master/exec.c#L63) 将命令行参数字符串逐个压入到用户堆栈中，并压入假的返回地址 `0xffffffff` 和参数个数 `argc`，最后调整堆栈指针。

### 1.4 切换进程空间

[exec.c#L96](https://github.com/professordeng/xv6-expansion/blob/master/exec.c#L96) 将对新进程 PCB 中有关进程空间的信息进行填写，通过 `switchuvm()` 完成页表的切换，从而完成进程空间的切换。此时新进程的入口地址 `proc->tf->eip` 设置为可执行文件的 入口地址（变量 `elf.entry`）。 

## 2. init

系统的第一个进程。

### 2.1 initcode.S

`initcode.S` 是 `init` 进程的前身，该程序功能就是通过 `exec` 系统调用来执行磁盘上的 `/init` 程序。

### 2.2 init.c

与正常的 Linux 程序不同，其命令行参数 `argv[]` 不是外部传入的，而是直接写在代码中的。 程序开始处显示打开 console 控制台设备作为 0 号文件，然后执行两次 `dup(0)` 使得 1 号、2 号文件也是 console 设备，它们分别对应 “标准输入”、“标准输出” 和 “标准出错” 文件。也就是说，后续的输入输出操作都是控制台的 `IO`。后面我们会看到控制台的输入是键盘或串口， 输出是 CGA 显卡或串口。 

从 `init.c` 的代码可以看出，`init` 进程是一个无限循环，创建 `sh` 进程并等待 `sh` 结束，如果 `sh` 进程结束，则 `for` 的下一次迭代会再次创建 `sh`。我们在 `xv6` 终端中用 `Ctrl + p` 查看当前 `sh` 进程号为 2，用 `kill 2` 命令将 `sh` 进程撤销，将会看到 `init` 执行 `for` 循环下一次迭代，打印出 `init: starting sh`，此时再用 `Ctrl + p` 查看发现新的 `sh` 进程号是 5。 

## 3. sh.c

这是 `xv6` 的 `shell` 程序，它的主函数在 [sh.c#L144](https://github.com/professordeng/xv6-expansion/blob/master/sh.c#L144)。

首先打开三个标准文件：标准输入、标准输出和标准出错文件，其文件描述符分别为 0、 1、2。但是其代码是打开文件 0/1/2/3 然后再关闭文件 3，读者可以尝试修改一下避免打开再关闭文件造成的时间浪费。 

然后是一个循环，不断读入命令行的命令并执行，除了 `cd` 命令单独处理外，其他的命令（含内部命令）由 `runcmd(parsecmd(buf))` 完成。其中 `parsecmd()` 将会把用户输入的命令字符串解析，并填写通用命令数据对象 `cmd` 结构体中的命令类型。

`xv6` 中的其他命令数据对象结构体 （`execmd`、`redircmd`、`pipecmd`、`listcmd`、`backcmd`）的第一个成员就是通用数据对象 `cmd` 的全部成员（仅有一个）。

虽然关于命令字符串的分析、重定向、管道、多命令列表等问题的处理非常繁杂，但是其最核心和重要的却是 `exec()`。 

## 4. xv6 测试（usertest.c）

[usertests.c](https://github.com/professordeng/xv6-expansion/blob/master/usertests.c) 源代码用于测试 `xv6` 的基本功能，涵盖进程管理、内存管理和文件管理等方面， 可以作为 `xv6` 系统编成的典范，通过分析这些测试代码可以了解到系统的运行概貌。`usertests.c` 大约有 1800 行源代码，根据需要自行阅读。

## 5. 用户进程的 ELF

从 ELF 文件可以看出 `ls` 程序是从 0 地址开始运行代码的，这个与 Linux 上的可执行文件并不相同。 

```bash
➜  xv6-expansion git:(dev) readelf -h _ls                
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          14124 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         16
  Section header string table index: 15
```

如下信息所示，`_ls` 可执行文件只有一个 ELF 段需要装入，而且代码和数据合并在一个段，因此其属性必须是可读 R、可写 W 和可执行 E。如果查看 `ls` 程序的 ELF 文件，可以发现该程序没有 `.data` 节，也就是说没有可以读写的全局变量。 

```bash
➜  xv6-expansion git:(dev) readelf -l _sh        

Elf file type is EXEC (Executable file)
Entry point 0x0
There are 2 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000080 0x00000000 0x00000000 0x01872 0x018f0 RWE 0x20
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .eh_frame .data .bss 
   01    
```

## 6. ULIB 库

`xv6` 没有实现 C 语言标准库，因此也不能使用相应的头文件，但是它实现了一个最基本的用户态库 ULIB。`Makefile` 中关于 ULIB 生成规则为 `ULIB = ulib.o usys.o printf.o umalloc.o`，包 含了基本的打印输出、系统调用的用户态函数等。严格来说 ULIB 并不是库（例如 Unix / Linux 中的 `*.a` 或 `*.so`），它只是若干函数组成的可重定位目标文件，最终通过静态链接进 `xv6` 的可执行文件中。

### 6.1 用户态内存管理（umalloc.c）

用户态的内存管理是通过 `umalloc.c` 实现的，用户态发出的各个内存分配请求，都使用一个 `header` 联合体来管理，具体包括起点指针和空间大小。而 `header` 联合体是直接嵌入在内存空闲区内部的。因此用户态的内存分配并不是以任意字节大小进行分配的，而是按照 `sizeof(header)` 的整数倍进行分配的，[umalloc.c#L69](https://github.com/professordeng/xv6-expansion/blob/master/umalloc.c#L69) 就是为了求得规整后的长度，以 `header` 大小计算的长度 `nunits`。

进程首次执行 `malloc()` 时，[*freep](https://github.com/professordeng/xv6-expansion/blob/master/umalloc.c#L22) 为空，因此执行 ，对 `*freep` 和 `base` 进行初始化。再分配出去之后，可用空间的起点应该要刨除 `sizeof(Header)` 之后的位置，同理回收的时候给出的地址指针要回退 `sizeof(Header)` 才能获得 `Header` 的位置。分配过程是在空闲区的链表中扫描，找到合适的空闲区，并修改链表。如果扫描后未能发现足够大的空间，则使用 `morecore()` 对堆进行扩展，最终将调用 `sbrk()` 系统调用。 

### 6.2 usys.S

用户态代码要进行系统调用时通过 `ulib` 库中的代码，它们是通过 [usys.S](https://github.com/professordeng/xv6-expansion/blob/master/usys.S) 经过变异后生成的 `usys.o` 并链接到 `ulib.o` 中的。`usys.S` 中定义了用户代码调用 `fork()`、`open()`、`read()` 等系统调用的 C 函数入口，这些入口函数内部将进一步使用 `int` 汇编指令通过软中断机制进入内核的系统调用处理函数。

### 6.3 ulib.c

[ulib.c](https://github.com/professordeng/xv6-expansion/blob/master/ulib.c) 是一些通用的函数，例如内存拷贝、字符串比较等操作。如果读者需要为 `xv6` 代码进行增强，那么一些比较通用的函数可以放到这个文件中并进入到 `ulib.o`，或者放到独立的 C 文件中最终进入到 ULIB 对象中。 

### 6.4 printf.c

[printf.c](https://github.com/professordeng/xv6-expansion/blob/master/printf.c) 是用户态代码调用的打印函数，注意区分与内核代码使用的 `cprintf()`、`panic()` 等函数。 用户态输出代码的核心是 `putc()` 函数，它可以向文件（例如控制台的显示器）中写入一个字节， 其他输出函数建立在 `putc()` 之上。 

 