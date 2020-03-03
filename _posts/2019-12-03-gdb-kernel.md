---
title: 3. 调试 kernel
---

之前学习完调试 `bootmain` 后，我们知道了 `bootmain` 为 kernel 做了以下准备工作：

1.  进入 32 位的保护模式。
2. 设置好段寄存器和 GDTR 寄存器，此时没有分段功能，逻辑地址就是线性地址。
3. 将 kernel 加载到 `0x100000` 开始的物理内存上。

接下来 `bootmain` 就把控制权交给了 kernel。所以这节我们观察 kernel 的初始过程。

ELF 的入口地址会记录在 ELF 头部，我们查看如下

```bash
# readelf -h kernel
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
  Entry point address:               0x10000c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          178184 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         18
  Section header string table index: 17
```

而我们知道 kernel 的入口函数正是 `entry.S` 中的 [entry()](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L45)。尝试给 `entry()` 打断点发现函数位于 `0x8010000c` 的位置，这是因为 `gdb` 显示的是逻辑地址，也是线性地址，但是此时还没有开启分页，物理地址是 `0x10000c`，因此这个断点是无效的。

```bash
(gdb) b entry
Breakpoint 1 at 0x8010000c: file entry.S, line 47.
```

所以我们只能直接用内存地址打断点，如下：

```bash
The target architecture is assumed to be i8086
[f000:fff0]    0xffff0:	ljmp   $0x3630,$0xf000e05b
0x0000fff0 in ?? ()
+ symbol-file kernel
(gdb) b *0x10000c
Breakpoint 1 at 0x10000c
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x10000c:	mov    %cr4,%eax

Thread 1 hit Breakpoint 1, 0x0010000c in ?? ()
(gdb) p/x $esp
$1 = 0x7bdc
```

执行过程目标架构也从 `i8086` 切换到 `i386`，接下来的 kernel 就是运行在 `i386` 上的。此时成功进入到 `entry()` 函数中。并且我们发现 `$esp` 从 `0x7c00` 下降到了 `0x7bdc`，也就是 `bootmain` 执行过程中栈发现了变化。

## 1. 开启大页模式

接下来通过将 `%cr4` 的 `CR4_PSE` 位置 1，开启大页模式。相关代码如下

```assembly
# Turn on page size extension for 4Mbyte pages
movl    %cr4, %eax
orl     $(CR4_PSE), %eax
movl    %eax, %cr4
```

执行前 `CR4=00000000`，我们打断点到 `0x100028`，按 `c` 继续：

```bash
(gdb) b *0x100028
Breakpoint 2 at 0x100028
(gdb) c
Continuing.
=> 0x100028:	mov    $0x8010b5c0,%esp

Thread 1 hit Breakpoint 2, 0x00100028 in ?? ()
```

此时 `CR4=00000010`。

设置为大页模式后，紧接着就开始设置大页模式下的页表 `entrypgdir[]`，通过将页表的物理地址赋给 `%cr3` 即可。赋值前 `CR3=00000000`，赋值后 `CR3=00109000`。也就是说 `entrypgdir[]` 的物理地址就是 `00109000`，这里用到了 [V2P_WO()](https://github.com/professordeng/xv6-expansion/blob/master/memlayout.h#L11) 宏将逻辑地址转为物理地址。

```assembly
# Set page directory
movl    $(V2P_WO(entrypgdir)), %eax
movl    %eax, %cr3
```

大页模式标志设置好，页表也设置好，此时可以开启硬件的分页机制了，通过设置 `%cr0` 的相关标志位，相关代码如下：

```assembly
# Turn on paging.
movl    %cr0, %eax
orl     $(CR0_PG|CR0_WP), %eax
movl    %eax, %cr0
```

设置前 `CR0=00000011`，这是由于 `bootmain` 设置过，设置后 `CR0=80010011`，主要用 `CR0_PG` 开启分页机制，`CR0_WP`（Write Protect）位置 1 是防止程序（包括内核代码自己）更改内核代码。

接下来开始设置栈指针 `%esp`，前一次设置是 `bootmain` 将其设置为 `0x7c00`。

```bash
(gdb) p/x $esp
$1 = 0x7bdc
(gdb) ni
=> 0x10002d:	mov    $0x80102eb0,%eax
0x0010002d in ?? ()
(gdb) p/x $esp
$2 = 0x8010b5c0
```

 [KSTACKSIZE](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L2) 为 `0x80000000`，因此 `stack` 位于物理地址 `0x10b5c0` 处，我们查看 kernel 的大小 为 `FileSiz=0x0a516`，对应物理地址 `0x10a516`，也就是说 `0x10a516~0x10b5c0` 为栈空间。

```bash
# readelf -l kernel 

Elf file type is EXEC (Executable file)
Entry point 0x10000c
There are 2 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x001000 0x80100000 0x00100000 0x0a516 0x154a8 RWE 0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .stab .stabstr .data .bss 
   01     
```

`%esp` 设置完毕，大页模式的 kernel 算是完成了。接下来就是跳到 `main` 函数执行了，这里用到了间接跳转指令，因为直接跳转的话会生成 PC 相对跳转，无法进入到 “高地址” 的 `0x8000-0000` 之上的内核空间。

由于 kernel 的代码使用的是逻辑地址，`$main` 的值为 `0x80102eb0`，将其值赋给 `%eax`，然后 `jmp` 指令跳转到 `%eax` 所指的地址处。由于已经启动了大页模式的分页机制，以后的逻辑地址都会经过分页机制转换为物理地址后再访问内存。

```assembly
# Jump to main(), and switch to executing at
# high addresses. The indirect call is needed because
# the assembler produces a PC-relative instruction
# for a direct jump.
mov $main, %eax
jmp *%eax
```

`$eip` 的前后变化如下，此后内核就工作在高地址上了。

```bash
(gdb) p/x $eip
$3 = 0x100032
(gdb) ni

Thread 1 received signal SIGTRAP, Trace/breakpoint trap.
=> 0x80103761 <mycpu+17>:	mov    0x80112d00,%esi
mycpu () at proc.c:48
48	  for (i = 0; i < ncpu; ++i) {
(gdb) p/x $eip
$4 = 0x80103761
```

