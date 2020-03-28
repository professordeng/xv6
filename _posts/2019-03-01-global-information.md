---
title: 1. 全局性信息
---

xv6 的系统启动过程和 x86 上的其他操作系统启动过程类似，都经历主板 BIOS 代码，转入启动扇区代码，然后转入内核启动代码，最后创建 `init` 以及 shell 进程。

下面先简单介绍一些包含全局性信息的代码，然后从 `bootloader` 的启动扇区代码 `bootblock` 开始，再进入的 xv6 操作系统的初始化代码。

**注意** ：xv6 源代码是操作系统代码，因此与硬件架构有密切关系。启动代码将完成 一些硬件初始化工作的细节。

全局性信息我们放在最开头，主要是方便读者查阅。类似地，代码分析的章节安排是从程序开机、然后执行引导程序、再进入 xv6 初始化的顺序进行。但是从学习的角度上说，读者并不需要完全按此章节次序学习（建议直接从 `bootblock` 开始学习，甚至可以先从大家比较熟悉的进程管理入手，然后在需要的时候再返回这里学习和查阅信息）。

 ## 1. 系统常数

在 [param.h](https://github.com/professordeng/xv6-expansion/blob/master/param.h) 文件中定义了几个常量，被后续的代码所引用。可以大致浏览一下，后面在用到相关常数的时候可以回来看看，各个常数的用途在代码的注释部分有简要的说明。

## 2. 硬件相关代码

在 [x86.h](https://github.com/professordeng/xv6-expansion/blob/master/x86.h) 文件中提供了若干用于访问硬件的内嵌汇编函数，以及因中断 / 系统调用而进入内核态时在堆栈中建立的 [trapframe](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L147)。

### 2.1 访问硬件（内嵌汇编）

该源文件用于封装 x86 汇编指令并形成内嵌（`inline`）函数，使得其他 C 代码可以方便地使用底层的硬件操作指令。例如 [inb()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L3) 用于从指定的 IO 端口（`port`）读入一个字节，[outb()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L21) 是向指定的 IO 端口（`port`）输出一个字符（`data`）。其他用于 IO 读写的函数还有 `insl()`、`outw()`、`outsl()`、`stosb()` 和 `stosl()`。

[lgdt()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L62) 用于设置 GDTR 寄存器，[lidt()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L76) 用于设置 IDTR 寄存器，[ltr()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L88) 用于设置 TR 寄存器。[loadgs()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L102) 用于设置附加段 GS 寄存器。

[cli()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L108) 用于关中断，[sti()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L114) 用于开中断。[readeflags()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L94) 用于读取表寄存器的值。[xchg()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L120) 用于实现原子性的交换操作，[rcr2()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L133) 用于读 `CR2` 寄存器内容，[lcr3()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L141) 用于设置页表寄存器 `CR3`。

除了前面的体系结构相关的操作外，还定义了一个 `trap` 的栈帧结构 `trapframe`，是进程进入内核态的时候在内核栈中构建的一个数据结构。`trapframe` 用于记录用户态断点的信息，其细节将在下面展开讨论。 

- 注意：有些读者可能并不熟悉里面的汇编格式。如果读者对 C 代码中使用内嵌汇编的疑问，暂时也不用纠结，只要了解其功能作用就不影响其他内容的学习。

### 2.2 trapframe

用户代码和内核代码是相互独立的执行流，用户代码的执行会被内核代码执行流所打断，此时用户进程的执行流断点现场将保存在一个称为 `trapframe` 的陷阱帧中。当读者分析中断处理代码和系统调用时，将会用到该知识。

从用户态切换到内核态前，我们需要在内核堆栈中保存用户态断点的一些现场信息，以便将来返回到用户态断点继续运行。硬件中断会在堆栈中压入 SS、ESP、EFLAGS、CS 和 IP 等几个寄存器的信息作为 `trapframe` 的起点，然后中断处理进入到公共入口 [alltraps](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L3) 之后，将继续建立 `trapframe`（除了硬件自动压入的一些寄存器外还将继续压入 DS、ES、FS 和 GS，然后用 `pushal` 指令又压入了 EAX、ECX、EDX、EBX、ESP、EBP、ESI 和 EDI。此时构造了完整的 `trapframe`，保存了用户态断点的所有信息（也就是恢复被中断的用户态代码所需的全部信息）。

在 [trapframe](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L147) 结构体中，需要注意的是结构体成员地址从前往后的地址从低地址逐渐往高地址布局的。

在系统调用过程中，通过调用 [trap](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L35) 函数，将进行任务分发（根据堆栈 `trapframe` 里的保存的中断号 `trapno` 转入到相应函数去处理）。

如果中断（或系统调用）是在内核态发生的，则 `trapframe` 顶部没有 SS（stack segment）和 ESP（stack pointer） 两项，直接从 EFLAGS 开始。

### 2.3 x86.h

[x86.h](https://github.com/professordeng/xv6-expansion/blob/master/x86.h) 文件定义了访问 x86 硬件端口、控制寄存器、开关中断、原子交换等操作的内联函数，具体操作的实现都是通过内嵌汇编实现的。

汇编 `cld` 清掉 EFLAGS 中 DF 位（即方向位），这表示用 ESI 和 EDI 给内存赋值的时候才用增加地址的方式，`=D` 代表把 `addr` 赋到 EDI 寄存器中，`=c` 代表把 `cnt` 的值赋到 CX 寄存器中，`a` 代表把 `data`  赋到 `ax` 中，`rep` 代表重复这个操作。

在 `x86.h` 的后面定义了陷阱帧 `trapframe` 结构体，用于保存中断时的现场。`trapframe` 的高地址中的 ESP 和 SS 仅用于从用户态中断的情况。 

## 3. 段描述符

启动扇区 `bootblock` 将启动 x86 保护模式，需要设置段描述符。在 `bootblock` 汇编代码阶段，段描述符的产生依赖于 [asm.h](https://github.com/professordeng/xv6-expansion/blob/master/asm.h) 中的 `SEG_NULLASM` 和 `SEG_ASM` 两个宏定义。而到后面 C 语言阶段后，段描述符的生成使用 [mmu.h](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h) 中的 `SEG` 宏。 

### 3.1 创建段描述符

用于创建 x86 段的相关宏定义，分别是全空的段描述符 `SGE_NULLASM`，以及可以定制的段描述符 `SEG_ASM`。这两个宏在 `bootloader` 中用于构建 GDT 表的内容，以便进入保护模式。 另外有关于段属性（标志位）的宏，例如 `STA_X` 用于设置段描述符中的可执行标志位。

`asm.h` 中定义了汇编代码创建段描述符所用的 [SEG_NULLASM](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L5) 和 [SEG_ASM](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L11) 两个宏，以及段类型属性位的辅助信息（最后三行）。 