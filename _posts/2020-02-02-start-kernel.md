---
title: 11. kernel 启动
date: 2019-04-21
---

内核启动前，启动扇区 `bootblock` 已经为它准备好了内存镜像，内核的代码和数据合并在一起构成一个 “段”。注意此时已经进入保护模式并开启了分段机制，但由于各个段描述符实际上设置为起点为 0，长度为 `0xFFFF-FFFF`，实际上是放弃了分段地址映射功 能，仅保留了 `ring0`~`ring3` 的分段保护功能。此时还未开启 `x86` 地址的分页功能。 

## 1. 启动分页

进入内核代码 `entry.S` 后，将尽快开启分页机制。其中 `entry.S` 用于主处理器，`entryother.S` 用于后续启动的其他处理器。

### 1.1 entry.S

[entry.S](https://github.com/professordeng/xv6-expansion/blob/master/entry.S) 是 `xv6 kernel` 的入口代码，它将进入到 kernel 的 `main()` 函数。前者是汇编代码，后者是 C 语言代码。

前面提到 `bootbloader` 把内核从硬盘导入到 `0x100000` 处，不放在 `0x0` 开始的地方是因为从 `640kb` 到 `0x100000` 开始的地方是被用于 `IO device` 的映射，所以为了保持内核代码的连续性，就从 `0x100000`（1 MB）的内存区域开始存放。不把内核导入到更高的物理地址是因为 PC 物理内存有多有少，而这个低端地址是各种档次 PC 都应该具有的空间，所以放在 `0x100000` 处显然是个不错的选择。

kernel 代码从 [entry.S#L47](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L47) 处开始运行，在第 40~45 行可以看到两个符号，`_start` 和 `entry`。前者用于未启动分页时的物理地址（`bootblock` 运行时还还未开启分页），因此使用的是 `_start` = `0x0010000c` 地址；后者用于 kernel 运行分页后的虚拟地址 entry = `0x8010000c`。`_start` 符号是通过 [V2P_WO](https://github.com/professordeng/xv6-expansion/blob/master/memlayout.h#L14) 宏将 entry 地址转换而来的。 

在 47~49 行代码中，我们首先设置 `CR4` 的 `PSE` 位为 1。`CR4` 的 `PSE` 是页大小扩展位，如果 `PSE` 等于 0 则每页的大小是 4 KB，如果 `PSE` 等于 1 则每页的容量可以是 4 MB 或 4 KB。使用大页模式时，只需要查询一级页表（页目录）即可查找到 4 MB 的页帧，此时的页目录项的第 7 位（即 `PS` 位）为 1。否则如果检测到一级页表项的 `ps=0`，则还需要经过 二级页表的转换才能找到 4 KB 的页帧。这里采用大页模式主要是出于编程简单的原因，不用处理二级页表映射。 

启动分页模式将分两步完成，第一步设置 `CR3` 页表基地址寄存器，第二步通过 `CR0` 启动分页机制。

第 50~52 行代码执行第一步，设置页表寄存器 `CR3`，将页表首地址 `entrypgdir`（物理地址） 写入到 `CR3` 寄存器。此时使用的页表 `entrypgdir` 在 [mian.c#L103](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L103) 声明为 `pde_t entrypgdir[NPDENTRIES]`，可容纳 `NPDENTRIES=1024` 项。当前页表 `entrydir` 中只定义了两项， 分别把虚拟地址 `0~4MB` 和 `KERNBASE(0x8000-0000)` ~ `KERNBASE + 4MB` 的空间映射到了 `0~4MB` 的物理地址，`0x8000-0000` 则是启动分页之后内核所在的虚拟地址，注意这两项转换成物理地址后都是指向同一个地方（即内核所在的物理内存空间）。 

第 53~56 行的代码执行第二步：启动分页模式，主要是置位 `CR0` 的 PG 位。因为我们不希望有人（包括内核代码自己）可以更改内核代码，所以把 WP （Write Protect）位置位。

第 59 行代码设置了堆栈指针 `esp` 为 `stack + KSTACKSIZE`，由于 `stack` 是在 68 行 `.comm` 声明的，共占 `KSTACKSIZE=4096` 字节，因此 `esp` 指向了该区间的地址上限（栈底）。键入 `readelf -s kernel | grep stack`，知道 `stack` 被安排在 `0x8010a5c0` 的位置。

上一次对 `esp` 的设置是在 [bootasm.S#L65](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L65)，但是自从上次设置至今，发生了 两次函数调用：汇编中 `call bootmain` 和 C 代码 `entry()`。我们用 `gdb` 调试这段代码，由于 kernel 是按照 `0x80100000` 地址链接的，此时内核代码还运行未分页时 `0x100000` 开始的低地址，因此无法直接用行号做断点，只能够用 `b *0x10000c` 定位到 entry 处，查看到 `esp` 已经从原来的 `0x7c00` 下移到 `0x7bd`。

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

再用 `b *0x100028` 将断点设置到 `esp` 赋值之前，可以看到 `esp` 将更新为 `0x8010b5d0`。`esp - stack=0x8010b5c0-8010a5c0=4096` 正好是堆栈的大小。 

```bash
(gdb) b *0x100028
Breakpoint 2 at 0x100028
(gdb) c
Continuing.
=> 0x100028:	mov    $0x8010b5c0,%esp

Thread 1 hit Breakpoint 2, 0x00100028 in ?? ()
```

反推回去，`stack` 符号位于未分页地址的 `0x0010a5c0` 处，紧靠在 kernel 结束位置 `0x0010a516` 之后。

最后 65~66 行跳转到 [main.c#L18](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L18) 的 `main()` 函数，开始执行保护模式、具有分页管理的初始化代 码。物理内存中的 kernel 被映射到虚存的两个区间，但是在跳转前仍在 `0~4MB` 地址上运行 （虽然堆栈 `esp` 已经使用高地址的那个页表项）。 我们继续用命令停止在 `jmp` 指令处，可以看出跳转的目的地址 `$main=0x80102eb0` 位于高地址那个页，而当前 `eip= 0x100032` 仍处于低地址那个页。 

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

 ### 1.2 entryother.S

在主 CPU 完成初始化之后，其他 CPU 也将开始进行初始化（通过对接收到的 `STARTUP IPI` 进行响应而完成启动的）。非启动 CPU 称为 AP，AP 启动时处于实模式，`CS:IP=0xXY00 : 0x0000` （地址低 12 bit 为 0，在 4096 字节边界对齐），其中 `XY` 是通过 `STARTUP IPI` 传递过来的一个 8 bit 数据。 

主处理器每次启动一个 `AP`，将 `entryother` 拷贝到 `0x7000` 地址处（前面），并在 `start(0x7000) - 4` 地址处存放该处理器核上调度器 `scheduler` 的私有内核堆栈地址，在 `start(0x7000) - 8` 的位置放入 `mpentry` 的跳转地址，在 `start(0x7000) - 12` 的地方存放（启动时临时）页表 `entrypgdir` 地址。 

其过程类似于主 CPU，也需要从 16-bit 实模式到 32-bit 保护模式的转换，然后用（start - 12） 地址单元的值设置页表寄存器并启动大页分页模式，用（start - 4）地址单元的值设置 `esp`，最后调用（start - 8）地址单元指向的函数 `mpentry()`，开始具有分页模式的初始化代码。

 ## 2. kernel 的 main()

kernel 将从汇编 `entry.S` 转入到 `main.c` 的 C 代码 `main()` 函数，完成若干项初始化工作，在 `userinit()` 创建第一个用户进程后进入 `mpmain()` 函数进行 `scheduler()` 调度。

下面对 `main()` 中的所调用的各种初始化函数按调用次序做一个简单的介绍，并且指出它们在哪里继续深入讨论。 

物理内存管理使用了链表将空闲页帧进行组织，`kinit2()` 用于初始化时将所有空闲页帧构成链表。 

1. `kinit1()` 将现有的 4 MB 物理页帧中未使用的部分，按照 4 KB 页帧的尺寸构成链表。
2. `kvmalloc()` 用于对内核空间创建页表，此时使用的 4 KB 页而不是使用原来的 4 MB 大页模式的页表，具体参见 [vm.c#L138](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L138)。
3. `mpinit()` 用于检测其他 CPU 核，并根据收集的信息设置 `cpus[ncpu]` 数组和 `ismp` 多核标 志。
4. `lapicinit()` 对本地 `apic` 中断控制器进行初始化。
5. `seginit()` 段表中的段描述符初始化，在原来的两个内核段的基础上增加了用户态段。
6. `picinit()` 对 `8259A` 中断控制器初始化代码（适用于单核系统）。
7. `ioapicinit()` 对 `I/O APIC` 中断控制器初始化（适用于多核 `SMP` 系统）。
8. `consoleinit()` 完成控制台初始化。
9. `uartinit()` 完成串口初始化。 
10. `pinit()` 通过 `initlock()` 完成进程表 `ptable` 的互斥锁的初始化（开锁状态）。
11. `tvinit()`对中断向量初始化，而真正将 `IDT` 装入到 `IDTR` 要等待 `mpmain()->idtinit()` 的时候。
12. `binit()` 初始化磁盘块缓存（buffer cache），完成互斥锁初始化（开锁状态），并将缓冲区构成链表。
13. `fileinit()` 通过 `initlock()` 完成对 `ftable` 的互斥锁进行初始化（开锁状态），`table` 记录系统所打开的文件，总数不超过 `NFILE=100`。
14. `ideinit()` 对 `IDE` 控制器进行初始化，包括互斥锁的初始化、相应的中断使能以及检测是否有 `driver1`。
15. 如果不是 `SMP` 多核系统（`!ismp`），则调用 `timerinit()` 设置单核系统的定时器。
16. `startohters()` 用于启动其他处理器核。
17. `kinit2()` 函数将 4 MB~240 MB 地址范围的空闲页帧（4 KB 大小的页帧）构成链表。
18. `userinit()` 在 [proc.c#L118](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L118) 中定义，用于创建第一个用户态进程 `init`。
19. `mpmain()` 将 `IDT` 表的起始地址装入到 `IDTR` 寄存器中，然后开始执行 `scheduler()` 内核执行流。

除了比较简单的初始化函数在下面讨论外。其他复杂的初始化函数将会在各自的子系统分析中加以讨论，例如 `kvmalloc()` 在内存管理子系统中讨论。 

### 2.1 seginit()

`seginit()` 定义于 [vm.c#L13](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L13)，每个处理器都要执行这个初始化，最终各个处理器核 上的段表内容几乎相同：在每 CPU 变量 `cpus[].gdt[]` 中设置 4 项，分别对应内核代码段 `SEG_KCODE`、内核数据段 `SEG_KDATA`、用户代码段 `SEG_UCODE`、用户数据段 `SEG_UDATA`。 

各处理器不同的地方是 `SEG_KCPU` 段，由段寄存器 `GS` 使用，对应于 “每 CPU” 变量。 

### 2.2 mpmain()

当主处理器或其他 AP 处理器启动到 `mpmain()`，说明已经是启动代码的最后阶段，随后将进入 `scheduler()` 调度器中的无限循环（不再返回）。因此，`mpmain()` 的第一件事情就是打印 `cpuX: starting` 的信息，然用 `idtinit()` 装入中断描述符 `IDT` 表以响应时钟中断（及其他中断）， 接着将 `cpu->started` 设置为 1，让其他处理器知道本处理器已经完成启动。最后进入到调度器的 `scheduler()` 中的无限循环中，每隔一个 `tick` 时钟中断就选取下一个就绪进程来执行。

### 2.3 mpenter()

其他 AP 处理器经历 `entryother.S` 启动代码后，进入到 `mpenter()`。`mpenter()` 用 `switchkvm()` 切换到本处理器内存空间；再用 `seginit()` 设置段表（各处理器段表几乎完全相同），除了 `GS` 使用的 `SEG_KCPU` 段用于访问 “每 CPU” 变量；在完成本地中断控制电路 `local APIC` 的初始化 `lapicinit()`，然后就可以进入到 `mpmain()` 执行最后阶段的代码完成启动并进入到调度器循环中。

### 2.4 startothers()

`startothers()` 先将 `entryother.S` 代码从现在的位置拷贝到 `0x7000` 物理地址，然后逐个 CPU 启动。每启动一个 CPU 核，需要为它传递独立的堆栈指针、`mpenter()` 地址和系统共享的页表地址 `entrypgdir`。 