---
title: 3. 内核空间
---

在共享的物理页帧基础之上，通过分页机制的页表实现 ”虚~实“ 地址转换，进而形成各个进程独立的虚存空间。 内核启动后则进入进程调度器 `schduler()` 的无限循环，因此 `sheduler()` 使用的页表所表示的虚存空间，就称为内核空间。在 kernel 的 `main()` 中调用 [kvmalloc()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L138) 来创建 scheduler 内核执行流所使用的页表 [kpgdir[]](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L143)（替换早期页表 [entrypgdir[]](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L51)），从而建立起 XV6 的内核态虚存空间。 这个页表内容也用来构建每个进程的内核空间，每个进程页表对应内核的部分（高地址部分）将拷贝 `kpgdir[]` 内容（与 scheduler 执行流相同），而对应用户空间的部分页表将根据应用程序的 ELF 文件而创建。 

## 1. 内核页表的初始化

XV6 的内核执行流，只使用内核空间的页表，在用户空间没有映射任何内容（即 `0x8000-0000` 以下地址所对应的页表项都没有映射到物理页帧。

### 1.1 scheduler 内核页表

XV6 kernel 在 `main()->init1()` 之后紧接着执行 `kvmalloc()` 建立内核空间。`kmalloc()` 是通过 `setupkvm()` 创建页表（4 KB 大小的页）并记录在全局变量 [kpgdir](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L11) 中，然后 `switchkvm()` 切换使用该页表（不再使用 `bootblock` 建立的只有两项的、4 MB 的大页的 `entrypgdir` 页表）。  

[setupkvm()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L117) 用于建立内核空间所对应的页表项，该函数先用 `kalloc()` 分配一个页帧作为页表， 然后依据 [kmap[]](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L103) 数组来填写 `kpgdir` 页表。

`kmap[]` 指出了内核中多个不同属性的区间。

1. `kmap[0]` 映射了物理内存低 1 MB 的空间。
2. `kmap[1]` 是内核代码及只读数据。
3. `kmap[2]` 是内核数据。
4. `kmap[3]` 映射了用于设备的空间。

XV6 内核空间采用的映射方式和 Linux 内核映射相似，称为直接映射（虚地址和物理地址之间恒定相差一个常数偏移），这样很容易在物理地址和内核虚地址之间进行转换。

[mappages()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L57) 对当前 `kmap[x]` 涉及的每一个页帧依次建立映射：通过 `walkpgdir()` 定位其 PTE 项，并按照 `kmap[]` 给出的映射要求填写该 PTE，从而完成 `kpgdir` 中所涉及的内核空间页表项的填写。传入的参数分别有：`pgdir`（页表首地址）、`va`（内核虚拟空间起始地址）、 `size`（地址区间大小）、`pa`（一个或多个连续物理页帧的起始物理地址）、`perm`（访问模式）。

### 1.2 页表项 PTE 查找

[walkpgdir()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L32) 用于查找页表项，可查找 X86 页表机制图阅读此函数。此处不能简单用 “页表” 来泛指，必须区分为 “页目录表” PD 和 “页表” PT 两级。

`walkpgdir()` 根据 PD 和 PT 查找给定虚地址 `va` 所对应的页表项 PTE。对于给定 `va` 虚地址，首先用 `PDX(va)` 获得页目录号（page directory index），[PDX](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L73) 宏实际上就是取 `va` 的高 12 位。

通过 PD 和 `PDX(va)` 索引可以找到 PDE（page directory entry），如果 PDE 指出对应的页表 PT 已经存在（相应标志位 `PTE_P` 为 1），则通过 `pgtab = (pte_t*)P2V(PTE_ADDR(*pde))` 获得 PT 的程序地址（虚地址），进而继续用 `PTX(va)` 获得 PTE 索引，最后通过 `&pgtab[PTX(va)]` 获得 PTE 的虚地址并返回。

如果 PDE 指出对应的页表 PT 不存在（相应标志位 `PTE_P` 为 0），则分成两种情况：

1. 当 `alloc==0` 则直接返回 0，表示没有找到对应的 PTE。一般用于内存访问时的地址转换。 
2. 如果 `alloc!=0` 则表示要分配 PT，并建立 PDE 到 PT 的映射。首先要分配一个页帧用于 PT，并将 PDE 指向该页帧。一般用于页表的创建和内存的分配时，建立地址映射时的操作。 

此处的 `PDX()` 和 `PTX()` 宏定义在 [mmu.h#L73](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L73)，各自从虚地址截取对应的字段。虚地址分解如下表：

| 10 bit               | 10 bit           | 12 bit             |
| -------------------- | ---------------- | ------------------ |
| Page Directory Index | Page Table Index | Offset within Page |
| `PDX(va)`            | `PTX(va)`        |                    |

 页表项 `PTE` 的内容如下表

| 20 bit                | 12 bit     |
| --------------------- | ---------- |
| Page physical address | Page flags |

### 2.3 各 CPU 的 GDT

在 kernel 的 `main()` 执行 `seginit()` 更新自己的全局段描述符 GDT 之前，主 CPU 使用的是 [bootasm.S#L80](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L80) 定义的三个段。但是当 `main()` 执行 `seginit()` 之后将重新设置段，此时使用的 GDT 表是每 CPU 变量 [c->gdt](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L18)，在执行 [vm.c#L25](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L25) 进行设设置， [vm.c#L29](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L29) 之后开始使用该 GDT。可以看出在地址映射方面，内核代码段 `SEG_KCODE`、内核数据段 `SEG_KDATA`、用户代码段 `SEG_UCODE` 和用户数据段 `SEG_UDATA` 都是从 0 地址到 `0xffff-ffff`（4 GB）的范围，也就是说从地址映射方面基本放弃了段式管理。但是各段的特权级是不同的，内核代码和数据特权级为最高 0，用户代码和数据为最低 `DPL_USER`（级别 3）。段定义均在 [mmu.h#L14](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L14)。

此时的段和原来相比，多了用户空间的两个段和每 CPU 变量所在的段。由于内核相关的段并没有变化，因此 `lgdt()` 并不影响 `seginit()` 的执行，并且正常返回到 `main()` 函数继续运行。 `main()` 后续还会执行 `init2()` 结束全部内存的初始化工作。  

 ## 2. 内核页表切换

内核 `scheduler()` 代码使用的页表建立后就不再变化，它映射了所有的物理页帧，因此可以读写全部物理页帧的内容。用户进程有自己的页表，在让出 CPU 而切换到内核 scheduler 执行流时才使用 [switchkvm()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L347) 切换到 scheduler 的页表。同理，从 scheduler 切换到下一个就绪进程使用 [switchuvm()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L343) 切换到用户进程页表。

### 2.1 页表切换与调度

当需要切换到 scheduler 的页表时，使用 `swtchkvm()` 即可，通过 `lcr3(V2P(kpgdir))` 完成， 反之从 scheduler 切换到用户进程的页表时，则使用 `swtchuvm()`，另外需要设置 TSS 段、用户态堆栈信息、最后才通过 `lcr3(V2P(p->pgdir))` 完成页表切换。从各个进程切换到内核态后，对应唯一的一个 “内核代码及其堆栈”，但是从内核态返回到用户态时，则并不唯一，因此必须设置 TSS 中的用户进程的堆栈位置。

在进程切换过程中，需要上述两种操作。例如进程 A 切换到进程 B 的过程如下：A 进程通过某种途径进入内核用 `swtchkvm()` 切换到 scheduler，然后再通过 `swtchuvm()` 切换到进程 B。

无论上述两个操作怎么切换，内核空间的映射关系总是相同的。也就是说用户进程的页表 `0x8000-0000` 以上地址的映射和 scheduler 使用的页表内容是一样的。用户进程切换到内核态 （例如系统调用或中断）时，虽然是使用自己的页表中的高地址部分，但是和 `scheduler()` 看到的空间是一样的。  

调用 `switchuvm()` 的地方有三处：`exec()` 创建新的进程镜像后、`scheduler()` 切换回用户进程时、`growproc()` 修改用户空间范围后（其实 `growproc()` 只是需要重新装载 `CR3` 寄存器，引起现有 TLB 作废即可，而无须完整的 `switchuvm()` 操作）。

### 2.2 TSS 与调度

当调用 [switchuvm()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L155) 从 scheduler 切换到用户进程时，并不作 Intel 概念上的任务切换。 `switchuvm()` 借助任务状态段 TSS 完成堆栈的切换，然后用 `lcr3()` 切换到进程页表，从而完成到用户进程空间的切换。`switchuvm()` 中的 `cpu->gdt[SEG_TSS].s=0` 将任务状态段设置为系统段， 还将 TSS 段的内核态堆栈段 `cpu->ts.ss0` 设置为 `SEG_KDATA` 段、内核堆栈指针 `cpu->ts.esp0` 初始化为切入进程的内核栈 `proc->kstack+KSTACKSIZE`。其中执行 `ltr(SEG_TSS<<3)` 的操作将 TR 寄存器指向 `cpu->ts`，实际上只需要执行一次即可，只不过 `switchuvm()` 并没有区分第一次和后续的调用，都反复设置 TR 寄存器为相同的 `SET_TSS<<3`。 

而 Linux 也采用了类似的实现方式，体现了软件对硬件体系结构的灵活应用。早期 Linux 内核有进程最大数的限制（受限于 GDT 表项的数量），受限制的原因是每一个进程都有自已的 TSS 和 LDT，而 TSS（任务描述符）和 LDT（私有描述符）必须放在 GDT 中，GDT 最大只能存放 8192 个描述符。从 Linux 2.4 以后，全部进程使用同一个 TSS，准确的说是每个 CPU 一个 TSS， 在同一个 CPU 上的进程使用同一个 TSS。Linux 2.4 以后的内核不再使用硬切换，而是使用软切换。寄存器不再保存在 TSS 中了，而是保存在 `task->thread` 中，只用 TSS 的 `esp0` 和 IO 许可位图。所以，在进程切换过程中，只需要更新 TSS 中的 `esp0`、`io bitmap`。任务切换（硬切换）需要用到 TSS 来保存全部寄存器（`2.4` 以前使用 `jmp` 来实现切换）。

## 3. 相关代码

[vm.c](https://github.com/professordeng/xv6-expansion/blob/master/vm.c)

`vm.c` 包含了内核空间和用户空间的管理代码，包括初始化操作和运行时的操作。 

[mmu.h](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h)

`mmu.h` 是与虚存地址管理相关的信息、CPU 现场 [taskstate](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L106) 以及 X86 的门描述符。

 #### 内存地址相关代码

[mmu.h#L4](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L4) 定义了 EFLAGS 标志寄存器的各个标志位。

[mmu.h#L7](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L7) 定义了 `CR0` 控制寄存器各个控制位。

[mmu.h#L12](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L12)  定义了 `CR4` 控制寄存器的 `PSE` 位。 

[mmu.h#L14](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L14) 给出了 XV6 所使用的段的编号。

包括段 0，一共有 [NSEGS](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L22) 个段。上述段的编号、名称和用途总结如下表。

| 段号 | 名称        | 用途       |
| ---- | ----------- | ---------- |
| 0    |             | 无效       |
| 1    | `SEG_KCODE` | 内核代码段 |
| 2    | `SEG_KDATA` | 内核数据段 |
| 3    | `SEG_UCODE` | 用户代码段 |
| 4    | `SEG_UDATA` | 用户数据段 |
| 5    | `SEG_TSS`   | 任务状态段 |

[mmu.h#L24](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L24) 定义了段描述符 `segdesc` 结构体。 

[mmu.h#L42](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L42) 定义了两个宏，用于按指定要求生成段描述符的变量声明。 

[mmu.h#L55](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L55) 给出了用户段的类型标志位和系统段的类型标志位。

[mmu.h#L65](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L65) 给出了虚地址的页目录索引、页表索引和页内偏移的划分情况，以及用于获取虚地址页目录索引的 `PDX()` 和页表索引的 `PTX()`。

[mmu.h#L82](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L82) 定义了页表相关的几个常量。

[mmu.h#L99](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L99) 的两个宏分别用于获取虚地址的页索引号 `PTE_ADDR(pte)` 和页的访问标记 `PTE_FLAGS(pte)`。 

地址操作的宏总结为以下几个： 

1. `PDX(va)` 宏用于取得虚地址的页目录索引值。
2. `PTX(va)` 宏用于获得虚地址的页表索引值。 
3. `PGADDR(d, t, 0)` 宏根据虚地址的目录索引、页表索引和页内偏移来构建虚拟地址的值。
4. `PTE_ADDR(pte)` 取得 PTE 高 20 位（对应的页帧的物理地址）。
5. `PTE_FLAGS(pte)` 取得 PTE 低 12 位（对应页访问标记）。

[mmu.h#L106](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L106) 给出了任务状态段 TSS 所对应的结构体 `taskstate`。

[mmu.h#L147](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L147) 给出了中断和陷阱对应的门描述符（`gatedesc` 结构体）的描述。

[mmu.h#L160](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L160) 的 `SETGATE` 宏则用于定制门描述符。

如果没有定义汇编 `__ASSEMBLER__` 则 [mmu.h#L55](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L55) 和 [asm.h#L16](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L16) 重复。