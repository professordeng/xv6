---
title: 5. 启动分页
---

内核启动前，启动扇区 `bootblock` 已经为它准备好了内存影像，内核的代码和数据合并在一起构成一个 “段”。注意此时已经进入保护模式并开启了分段机制，但由于各个段描述符实际上设置为起点为 0，长度为 `0xFFFF-FFFF`，实际上是放弃了分段地址映射功 能，仅保留了 ring 0~3 的分段保护功能。此时还未开启 X86 地址的分页功能。 

进入内核代码 `entry.S` 后，将尽快开启分页机制。其中 `entry.S` 用于主处理器，`entryother.S` 用于后续启动的其他处理器。

## 1. entry.S

[entry.S](https://github.com/professordeng/xv6-expansion/blob/master/entry.S) 是 XV6 kernel 的入口代码，它将进入到 kernel 的 [main()](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L14) 函数。

前面提到 `bootblock` 把内核从硬盘导入到 `0x100000` 处，不放在 `0x0` 是因为 `0xa0000:0x100000` 开始的地方是被用于 IO device 的映射，所以为了保持内核代码的连续性，就从 `0x100000` 的内存区域开始存放。不把内核导入到更高的物理地址是因为 PC 物理内存有多有少，而这个低端地址是各种档次 PC 都应该具有的空间，所以放在 `0x100000` 处显然是个不错的选择。

kernel 代码从 [entry.S#L47](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L47) 处开始运行，可以看到有两个符号 [_start](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L40) 和 [entry](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L44)。前者用于未启动分页时的物理地址（`bootblock` 运行时还还未开启分页），因此使用的是 `_start` = `0x0010000c` 地址；后者用于 kernel 运行分页后的虚拟地址 entry = `0x8010000c`。`_start` 符号是通过 [V2P_WO](https://github.com/professordeng/xv6-expansion/blob/master/memlayout.h#L14) 宏将 entry 地址转换而来的。 

在 [entry.S#L46](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L46) 我们首先设置 `CR4` 的 PSE 位为 1。`CR4` 的 PSE （page size extension）是页大小扩展位，如果 PSE 等于 0 则每页的大小是 4 KB，如果 PSE 等于 1 则每页的容量可以是 4 MB 或 4 KB。使用大页模式时，只需要查询一级页表（页目录）即可查找到 4 MB 的页帧，此时的页目录项的第 7 位（即 PS 位）为 1。否则如果检测到一级页表项的 PS=0，则还需要经过二级页表的转换才能找到 4 KB 的页帧。这里采用大页模式主要是出于编程简单的原因，不用处理二级页表映射。 

启动分页模式将分两步完成，第一步设置 `CR3` 页表基地址寄存器，第二步通过 `CR0` 启动分页机制。

[entry.S#L50](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L50) 执行第一步，设置页表寄存器 `CR3`，将页表首地址 `entrypgdir`（物理地址） 写入到 `CR3` 寄存器。此时使用的页表 [entrypgdir](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L103) 可容纳 [NPDENTRIES](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L83) 项。当前页表 `entrypgdir` 中只定义了两项， 分别把虚拟地址 `[0, 4MB)` 和 `[KERNBASE, KERNBASE+4MB)` 的空间映射到物理空间 `[0, 4MB)`，`0x8000-0000` 则是启动分页之后内核所在的虚拟地址，注意这两项转换成物理地址后都是指向同一个地方（即内核所在的物理内存空间）。 

[entry.S#L53](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L53) 执行第二步：启动分页模式，主要是置位 `CR0` 的 PG 位。因为我们不希望有人（包括内核代码自己）可以更改内核代码，所以把 WP（Write Protect）位置位。

[entry.S#L58](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L58) 设置了堆栈指针 ESP 为 `stack+KSTACKSIZE`，由于 `stack` 是 [.comm](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L68) 声明的，共占 [KSTACKSIZE](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L2) 字节，因此 ESP 指向了该区间的地址上限（栈底）。

键入 `readelf -s kernel | grep stack`，知道 `stack` 被安排在 `0x8010a5c0` 的位置。

上一次对 ESP 的设置是在 [bootasm.S#L65](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L65)，但是自从上次设置至今，发生了 两次函数调用 [call bootmain](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L66) 和 [entry()](https://github.com/professordeng/xv6-expansion/blob/master/bootmain.c#L47)。我们用 GDB 调试这段代码，由于 kernel 是按照 `0x80100000` 地址链接的，此时内核代码还运行在未分页时 `0x100000` 开始的低地址，因此无法直接用行号做断点，只能够用 `b *0x10000c` 定位到 entry 处，查看到 ESP 已经从原来的 `0x7c00` 下移到 `0x7bd`。

```bash
(gdb) b *0x10000c
Breakpoint 1 at 0x10000c
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x10000c:	mov    %cr4,%eax

Thread 1 hit Breakpoint 1, 0x0010000c in ?? ()
(gdb) p $esp
$1 = (void *) 0x7bdc
```

再用 `b *0x100028` 将断点设置到 ESP 赋值之前，可以看到 ESP 将更新为 `0x8010b5d0`。`esp - stack=0x8010b5c0-8010a5c0=4096` 正好是堆栈的大小。 

```bash
(gdb) b *0x100028
Breakpoint 2 at 0x100028
(gdb) c
Continuing.
=> 0x100028:	mov    $0x8010b5c0,%esp

Thread 1 hit Breakpoint 2, 0x00100028 in ?? ()
```

其中 `0x8010b5c0` 就是指代  `stack+KSTACKSIZE` ，而 [KSTACKSIZE](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L2) 为 `0x1000`，可得 `stack` 符号位于未分页地址的 `0x0010a5c0` 处，紧靠在 kernel 结束位置 `0x0010a516` 之后。

[entry.S#L65](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L65) 跳转到 [main()](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L18)  函数，开始执行保护模式、具有分页管理的初始化代码。物理内存中的 kernel 被映射到虚存的两个区间，但是在跳转前仍在 `[0, 4MB]` 地址上运行 （虽然堆栈 ESP 已经使用高地址的那个页表项）。 我们继续用命令停止在 `jmp` 指令处，可以看出跳转的目的地址 `$main=0x80102eb0` 位于高地址那个页，而当前 `eip= 0x100032` 仍处于低地址那个页。 

```bash
(gdb) ni
=> 0x10002d:	mov    $0x80102eb0,%eax
0x0010002d in ?? ()
(gdb) ni
=> 0x100032:	jmp    *%eax
0x00100032 in ?? ()
(gdb) p $eip
$2 = (void (*)()) 0x100032
```

这里使用了间接跳转，因为直接跳转的话会生成 PC 相对跳转，无法进入到 “高地址” 的 `0x8000-0000` 之上的内核空间。

## 2. entryother.S

在主 CPU 完成初始化之后，其他 CPU 也将开始进行初始化（通过对接收到的 `STARTUP IPI` 进行响应而完成启动的）。非启动 CPU 称为 AP，AP 启动时处于实模式，`CS:IP=0xXY00:0x0000` （地址低 12 bit 为 0，在 4096 字节边界对齐），其中 `XY` 是通过 `STARTUP IPI` 传递过来的一个 8 bit 数据。 

主处理器每次启动一个 AP，将 `entryother` 拷贝到 `0x7000` 地址处，并在 `start(0x7000)-4` 地址处存放该处理器核上调度器 `scheduler` 的私有内核堆栈地址，在 `start(0x7000)-8` 的位置放入 `mpentry` 的跳转地址，在 `start(0x7000)-12` 的地方存放（启动时临时）页表 `entrypgdir` 地址。 

其过程类似于主 CPU，也需要从 16-bit 实模式到 32-bit 保护模式的转换，然后用 `$(start-12)` 设置页表寄存器并启动大页分页模式，用 `$(start-4)`设置 ESP，最后调用 `(start-80)` 指向的函数 [mpentry()](https://github.com/professordeng/xv6-expansion/blob/master/entryother.S#L72)，开始执行具有分页模式的初始化代码。
