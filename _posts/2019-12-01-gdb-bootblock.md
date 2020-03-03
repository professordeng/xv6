---
title: 1. 调试 bootblock
---

XV6 运行在 `i386` 处理器上（32 位 X86 处理器），在电脑刚启动的时候，首先执行的代码是主板上的 BIOS （Basic Input Output System）。BIOS 存放在非易失存储器中，主要完成一些硬件自检的工作。在这些工作做完之后，BIOS 会从启动盘里读取第一个扇区（启动扇区 boot sector）的 512 字节数据到内存中，这 512 字节的代码就是我们熟知的 `bootloader` 。在导入完成后 CPU 控制权由 BIOS 转交给 `bootloader` 。BIOS 会把 `bootloader` 导入到 `0x7c00` 开始的地方，然后把 PC 指针设成此地址（通过设置寄存器 `%ip`），将控制权交给 `bootloader` 。

这节我们通过跟踪 `bootloader` 的执行过程来观察 `bootbloader` 是如何工作的。

XV6 的 `bootloader` 是 `bootblock`，也是 `bootblock.o` 目标文件的代码节 `.text`，所以我们调试 `bootblock.o` 即可（单是 `.text` 的话 GDB 识别不了）。

一个终端启动 `make qemu-nox-gdb`（启动前需要修改 `.gdbinit` 的 `symbol-file`），另一个终端启动 `gdb bootblock -silent`。`-silent` 是不输出一些版本信息而已，可以不加。

查看是否将源代码读入进来：

```bash
(gdb) l
1	#include "asm.h"
2	#include "memlayout.h"
3	#include "mmu.h"
4	
5	# Start the first CPU: switch to 32-bit protected mode, jump into C.
6	# The BIOS loads this code from the first sector of the hard disk into
7	# memory at physical address 0x7c00 and starts executing in real mode
8	# with %cs=0 %ip=7c00.
9	
10	.code16                       # Assemble for 16-bit mode
(gdb)         # 回车重复上条指令
11	.globl start
12	start:
13	  cli                         # BIOS enabled interrupts; disable
14	
15	  # Zero data segment registers DS, ES, and SS.
16	  xorw    %ax,%ax             # Set %ax to zero
17	  movw    %ax,%ds             # -> Data Segment
18	  movw    %ax,%es             # -> Extra Segment
19	  movw    %ax,%ss             # -> Stack Segment
20	
```

## 1. BIOS 运行前

此源文件是 [bootblock.S](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S)，说明 BIOS 完成自检后就开始执行 `bootblock.S`，开始运行 BIOS 前我们先看看 `%cs`（代码段偏移） 和 `%eip`（指令指针） 的值，由于 `bootblock` 是 `.text` 代码节，所以只有代码和一些只读变量，所以也就还不需要数据段和堆栈。 

```bash
(gdb) p/x $cs   # p 表示输出，/x 表示十六进制格式
$1 = 0xf000
(gdb) p/x $eip
$2 = 0xfff0
```

## 2. bootblock 运行前

我们将断点设置在 `bootblock` 的第一条指令处，也就是 BIOS 执行后，`bootblock` 开始执行前，运行后再查看 `%cs` 和 `%eip` 的值。

```bash
(gdb) b start
Breakpoint 1 at 0x7c00: file bootasm.S, line 13.
(gdb) c
Continuing.
[   0:7c00] => 0x7c00 <start>:	cli    

Thread 1 hit Breakpoint 1, start () at bootasm.S:13
13	  cli                         # BIOS enabled interrupts; disable
(gdb) p/x $cs
$3 = 0x0
(gdb) p/x $eip
$4 = 0x7c00
```

此时正是 BIOS 刚跳转到 `bootblock` 的第一条指令而将控制权转交给 `bootblock` 的时刻。

## 3. bootblock.S 运行过程

BIOS 完成硬件自检后，将 `bootblock` 加载到 `0x7c00` 的物理地址处，就将控制权交给了 `bootblock`（通过设置 `$cs` 和 `$eip`）

接下来我们一步一步地追踪 `bootblock` 的运行过程，观察 `bootblock` 都做了哪些工作。`bootblock` 是运行在实模式的，逻辑地址、线性地址和物理地址均一致（此时没有分段和分页机制）。首先执行的是 `bootblock.S` 汇编代码，主要完成 3 项工作：

1. 切换到 32 位保护模式，一开始代码还是运行在 16 位模式下，这是由 [.code16](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L10) 决定的。
2. 跳到 `bootblock.c` 中执行相关代码

`bootblock.S` 执行的第一条指令就是关中断 `cli`，因为 BIOS 之前打开了，而 `bootblock` 是不允许外部中断的（除非自己代码中断或崩溃），因为 `bootblock` 的任务仅仅是把 kernel 的环境搭建好，开中断既不合理也不安全。

接下来设置三个段寄存器（ BIOS 已经设置了 `%cs`）的偏移地址为 0，相关代码如下，这说明此时没有逻辑地址和线性地址之分，逻辑地址就是线性地址。

在 16 位模式下，线性地址 = 段寄存器左移 4 位 + 逻辑地址（segment << 4 + offset），例如代码段的逻辑地址是 `0x1234`，代码段寄存器 `%cs` 为 `0x1234`，则线性地址为 `0x12340 + 0x1234`。所以此时逻辑地址是 16 位，但线性地址是 20 位，也就是说需要 20 根物理地址线，此时支持 1 MB 的物理空间。

```assembly
# Zero data segment registers DS, ES, and SS.
xorw    %ax,%ax             # Set %ax to zero
movw    %ax,%ds             # -> Data Segment
movw    %ax,%es             # -> Extra Segment
movw    %ax,%ss             # -> Stack Segment
```

由于 `segment:offset` 可以为 21 位地址，所以随后开启物理地址线 A20，这样就有 21 根物理线可用，具体是通过设置 `0x64` 端口和 `0x60` 端口，这样的话物理空间就增加到 2 MB。

```assembly
 # Physical address line A20 is tied to zero so that the first PCs 
  # with 2 MB would run software that assumed 1 MB.  Undo that.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```

接下来为进入保护模式做准备，使用一个 GDT 让虚拟地址直接映射到物理地址，以便有效内存映射在转换期间不会更改。

GDT 是一张存储在内存中的表，每个表项如下：

| Base（32 位） | Limit（20 位） | Flags（12 位） |
| ------------- | -------------- | -------------- |
| 基地址        | 段长           | 段的访问权限   |

为了访问这个表，IA32 专门设计了段寄存器 GDTR，总共 48 位，高 32 位为全局描述符表 GDT 的基址，低 16 位存放 GDT 限长。这里将 [gdtdesc](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L85) 变量赋给了 GDTR 寄存器。

```bash
gdtdesc:
  .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

相关代码如下：

```assembly
# Switch from real to protected mode.  Use a bootstrap GDT that makes
# virtual addresses map directly to physical addresses so that the
# effective memory map doesn't change during the transition.
lgdt    gdtdesc
movl    %cr0, %eax
orl     $CR0_PE, %eax
movl    %eax, %cr0
```

执行上面 4 条指令之前，在 `QEMU` 窗口的调试器下，`Ctrl + a` 、`c`，然后用 `info registers` 查看寄存器 GDT 的值为 0，`gdb` 窗口下执行 `s`，再次从 `QEMU` 窗口查看 GDT 的值如下： 

```bash
GDT=     00007c60 00000017
```

接下来的 3 步其实就是将 `%cr0` 的 `CR0_PE` 位置为 1，由于不能直接修改 `%cr0` 的值，所以借用了 `%eax` 这个通用寄存器。修改之前 `CR0=00000010`，修改之后 `%CR0=00000011`，从而开启保护模式。

最后通过 `ljmp` 的长跳转指令，重新加载 `%cs` 和 `%ip` 完成 32 位保护模式的转换，也就是 `i8086` 到 `i386` 的切换。

```bash
ljmp    $(SEG_KCODE<<3), $start32  # ljmp 段选择子, 段内偏移
```

加载之前我们记录一下寄存器的值如下：

```bash
(qemu) info registers
EAX=00000011 EBX=00000000 ECX=00000000 EDX=00000080
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00006f04
EIP=00007c2c EFL=00000006 [-----P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
CS =0000 00000000 0000ffff 00009b00 DPL=0 CS16 [-RA]
SS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
DS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
FS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
GS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     00007c60 00000017
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
DR0=00000000 DR1=00000000 DR2=00000000 DR3=00000000 
DR6=ffff0ff0 DR7=00000400
EFER=0000000000000000
FCW=037f FSW=0000 [ST=0] FTW=00 MXCSR=00001f80
FPR0=0000000000000000 0000 FPR1=0000000000000000 0000
FPR2=0000000000000000 0000 FPR3=0000000000000000 0000
FPR4=0000000000000000 0000 FPR5=0000000000000000 0000
FPR6=0000000000000000 0000 FPR7=0000000000000000 0000
XMM00=00000000000000000000000000000000 XMM01=00000000000000000000000000000000
XMM02=00000000000000000000000000000000 XMM03=00000000000000000000000000000000
XMM04=00000000000000000000000000000000 XMM05=00000000000000000000000000000000
XMM06=00000000000000000000000000000000 XMM07=00000000000000000000000000000000
```

然后在 `gdb` 调试窗口按 `s`  进入下一步，重新查看寄存器值的变化

```bash
(qemu) info registers
EAX=00000011 EBX=00000000 ECX=00000000 EDX=00000080
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00006f04
EIP=00007c31 EFL=00000006 [-----P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
CS =0008 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
SS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
DS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
FS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
GS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     00007c60 00000017
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
DR0=00000000 DR1=00000000 DR2=00000000 DR3=00000000 
DR6=ffff0ff0 DR7=00000400
EFER=0000000000000000
FCW=037f FSW=0000 [ST=0] FTW=00 MXCSR=00001f80
FPR0=0000000000000000 0000 FPR1=0000000000000000 0000
FPR2=0000000000000000 0000 FPR3=0000000000000000 0000
FPR4=0000000000000000 0000 FPR5=0000000000000000 0000
FPR6=0000000000000000 0000 FPR7=0000000000000000 0000
XMM00=00000000000000000000000000000000 XMM01=00000000000000000000000000000000
XMM02=00000000000000000000000000000000 XMM03=00000000000000000000000000000000
XMM04=00000000000000000000000000000000 XMM05=00000000000000000000000000000000
XMM06=00000000000000000000000000000000 XMM07=00000000000000000000000000000000
```

可以发现 `%cs` 从 `0000 00000000 0000ffff` 变为 `0008 00000000 ffffffff`，`%eip` 从 `00007c2c` 变为 `00007c31`。

我们用 `list *0x7c31` 查看跳转到的位置正好是 `.start32` 下的第一条指令。我们继续往下执行，首先设置所有的段寄存器（除了 `%cs`），此时已经是保护模式了，所以段寄存器存的是 GDT 的偏移。首先我们得看看

```assembly
# Set up the protected-mode data segment registers
movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
movw    %ax, %ds                # -> DS: Data Segment
movw    %ax, %es                # -> ES: Extra Segment
movw    %ax, %ss                # -> SS: Stack Segment
movw    $0, %ax                 # Zero segments not ready for use
movw    %ax, %fs                # -> FS
movw    %ax, %gs                # -> GS
```

这里的 [SEG_KDATA](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L16) 是 2，由于每个段描述符在 GDT 中占用 8 个字节，故 `%ds`、`%es`、`%ss` 均被设置为 16（2×8），而 `%fs` 和 `%gs` 暂时还不需要使用，故设置为 0。

最后在调用 `bootmain` 的 C 程序之前的最后一个步骤是在空闲内存中建立一个栈。内存 0xa0000 到 0x100000 属于设备区，而 XV6 内核则是放在 0x100000 处。引导加载器自己是在 0x7c00 到 0x7d00。本质上来讲，内存的其他任何部分都能用来存放栈。引导加载器选择了 0x7c00（即 [$start](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L12)）作为栈顶；栈从此处向下增长，直到 0x0000，不断远离引导加载器代码。

```bash
(gdb) p/x $esp
$13 = 0x6f04
(gdb) s
=> 0x7c48 <start32+23>:	call   0x7d3b <bootmain>
66	  call    bootmain
(gdb) p/x $esp
$14 = 0x7c00
```

所以物理内存 0~0x100000 的布局是

| 0~0x7c00       | 0x7c00~0x7d00    | 0xa0000~0x100000 |
| -------------- | ---------------- | ---------------- |
| 引导加载器的栈 | 引导加载器的代码 | 设备区           |

## 2. 实模式和保护模式

保护模式要比实模式的工作方式灵活许多。

1. 在实模式下，机器处于 16 位状态，此时线性地址 = 段寄存器 << 4 + 逻辑地址（`segment:offset`）。因此段基址必须是 16 的整数倍。

2. 在保护模式下，机器处于 32 位状态，此时线性地址（`segment:offset`）和逻辑地址的计算和实地址不太一样，段寄存器中的值将作为偏移（selector），代表这个段的段表项在 GDT / LDT 表中的索引。比如你当前要访问的地址是 `segment:offset` = `0x01:0x0000ffff`，此时由于每个段表项的长度为 8（64 位），所以此时应该取出地址 8 处的段表项。然后首先根据 Flags 字段来判断是否可以访问这个段的内容，这样做是为了能够实现进程间地址的保护。如果能访问，则把 `Base` 字段的内容取出，直接与 `offset` 相加，就得到线性地址（linear address）了。之后就是要根据是否有分页机构来进行地址转换了。

   由于 offset 是 32 位，所以保护模式下段的大小可以为 4 GB。

3. 实模式下段的长度是  64 B（offset 为 16 位逻辑地址），但是保护模式下段的长度可以达到 4 GB的。

4. 保护模式下可以对内存的访问多加一层保护，但是实模式没有。

## 3. GDT

`bootblock.S` 初始化了 GDT 表，相关代码如下：

```bash
# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULLASM                             # null seg
  SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
  SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg
```

这里要求 4 字节（32 位）对齐，有 3 个段表项，第一个表项使用 [SEG_NULLASM](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L5) 宏来生成，而后面两项用 [SEG_ASM](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L11) 宏来生成。

1. 第一项用 `SEG_NULLASM` 宏生成的项是全 0。
2. 二项对应 `bootblock` 代码段，起点为 `0x0`，长度为 `0xffffffff`，类型为 `STA_X|STA_R`（可执行可读）
3. 第三项对应 `bootblock` 数据段，起点为 `0x0`，长度为 `0xffffffff`，类型为 `STA_W`（可写）

## 4. 函数返回

最后加载器调用 C 函数 [bootmain](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L66)。`bootmain` 的工作就是加载并运行内核。只有在出错时该函数才会返回，这时它会向端口 [0x8a00](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L70) 输出几个字。在真实硬件中，并没有设备连接到该端口，所以这段代码相当于什么也没有做。如果引导加载器是在 PC 模拟器上运行，那么端口 0x8a00 则会连接到模拟器并把控制权交还给模拟器本身。无论是否使用模拟器，这段代码接下来都会执行 [spin](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L75) 死循环。而一个真正的引导加载器则应该会尝试输出一些调试信息。

例如后面 `bootmain()` 读入磁盘数据后发现不是 ELF 格式（即没有发现有效的内核）。这些代码执行两个 `outw` 指令的 IO 操作，向 `0x8a00` 端口写入数据从而引发 `Bochs` 虚拟机的 `breakpoint`（真实机器在 `0x8a00` 地址只是普通内存没有任何特定作用），然后进入无限循环。

至此，结束了 `bootasm.S` 中汇编代码的分析，开始第二步骤 C 语言的 `bootmain()` 代码运行阶段。

