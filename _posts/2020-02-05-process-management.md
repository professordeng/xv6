---
title: 1. 进程管理
date: 2019-06-01
---

进程是操作系统的核心概念，通过进程抽象使得一个物理机（处理器核）被虚拟化成多个，每个进程可以独立拥有一个完整的处理机运行环境。进程抽象依赖于运行环境的切换：CPU 现场的保存与恢复。因此进程切换代码与处理器架构密切相关，也比较难以理解。我们先从进程管理角度分析 `xv6`（这非常容易理解），然后再讨论进程调度中的切换细节问题。

## 1. 调度状态与执行现场

每次时钟中断都将进入到 `xv6` 内核代码，具体是 `trap()` 函数内部，完成 `IRQ_TIMER` 相关的处理（例如 `ticks++`），然后用 `yield()` 让当前进程让出 CPU（切换到其他就绪进程），具体参见 [trap.c#L103](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L103)。

### 1.1 进程控制块 PCB

进程描述符 PCB 在 `xv6` 中是 `proc` 结构体，具体定义请参见 [proc.h#L37](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L37)。proc 结构体记录了： 

1. 有关进程组织的信息：`pid`（进程号）、`parent`（父进程）、`name[]`（进程名）。
2. 进程运行状态信息：`state`（进程的运行调度状态）、`killed`（被撤销标志）。
3. 进程调度切换信息：`tf`（陷阱帧）、`context`（进程的上下文），`chan`（阻塞链表）。
4. 进程的内存映像的信息：`sz`（内存空间大小）、`pgdir`（页表）、`kstack`（内核栈底，创建进程 PCB 时由 `allocproc()->kalloc()` 分配）。
5. 文件系统相关信息：`cwd`（当前工作目录）、`ofile[]`（已打开文件的列表）。

从这里可以看到 `xv6` 的进程亲缘关系组织比 Linux 要简单得多，只有父进程关系，无法知道自己的子进程和兄弟进程。

进程的调度状态在 [proc.h#L35](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L35) 的枚举类型 `procstate` 中列出，分别是 `UNUSED`（未使用的 PCB）、`EMBRYO` （创建中，胚胎状态）、`SLEEPING`（睡眠阻塞中）、`RUNNABLE`（就绪状态）、 `RUNNING`（在 CPU 上运行）以及 `ZOMBIE`（僵尸态）。  

后面我们会看到，`xv6` 使用固定大小的静态数组来记录 PCB，因此未使用的 PCB 必须标为 `UNUSED`，而且系统中创建的进程数受限于该数组的大小。而 Linux 系统则是动态生成 PCB， 并构成链表，因此进程撤销后最终连 PCB 也释放掉而无须 UNUSED 状态，而且进程数目的不会受限于某个静态数组的大小。 

![process states](/xv6-book/img/state.png)

### 1.2 处理器组织管理

由于系统中有多个处理器，因此处理器的数量由全局变量 `ncpu` 记录，其声明 [proc.h#L14](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L14)（其定义在 [mp.c#L15](https://github.com/professordeng/xv6-expansion/blob/master/mp.c#L15)）。有一个全局数组变量 `cpus[NCPU]` 记录所有 CPU 的信 息，由于每个处理器对应的独立元素，因此只要保证各自只访问自己的元素，就不需要进行互斥保护（访问速度比自旋锁保护的变量更快速），这一类变量称为 "每处理器变量“（per-CPU 变量），我们也称它为 ”CPU 私有变量“。

系统中的处理器 `i` 通过 `cpus[i]` 来管理，这是一个 `cpu` 类型结构变量（见 [proc.h#L1](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L1)），用于该处理器上的进程调度管理。相关成员及其作用说明如下：

1. `apicid` 记录本地 `Local APIC ID`（`Intel` 的每个 CPU 都有独立的 `lapic`，每个 `lapic` 有一个 ID，`apicid` 是区分 CPU 的重要标识）。
2. scheduler 指向一个 context 结构体，context 对应于运行在本处理器上的 scheduler 执行现场（`edi` / `esi` / `ebx` / `ebp` / `eip`）。
3. `ts` 是 `taskstate` 结构体（参见 [mmu.h#L106](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L106) ），即 `x86` 的任务状态段的内容，从中可以找到因运行级提升所需的堆栈（例如中断进入内核态，需要切换到内核堆栈）。
4. `gdt[NSEGS]` 是本处理器正在使用的全局描述符表（各处理器差异在于 CPU 私有段）。
5. `started` 表示该处理器是否已经启动，多核系统刚开始有一个引导处理器（核）启 动，然后再其他其他处理器核。
6. `ncli` 记录了关中断的嵌套次数。
7. `intena` 记录了 `pushcli` 之前中断是否使能（打开）。
8. `cpu` 成员指向一个 `cpu` 结构体。  
9. `proc` 成员记录本 CPU 上正在执行的进程。

### 1.3 内核切换现场

我们在 [执行断点](https://neuron.zone/xv6-book/2019/04/11/kernel.html#执行断点) 中已经了解过 ”系统调用/中断断点“ 和 ”切换断点“。这里所谓的内核执行线程，则是内核态 ”切换断点“ 的执行现场。

在进程切换前要保存当前进程的内核态执行现场，并恢复切入进程的执行现场。由于进程切换都是发生在内核态的，而内核态的段寄存器都是相同的（CS 等等），因此无需保存段寄存器。而 `EAX`、`ECX` 和 `EDX` 也不需要保存，因为按照 `x86` 的调用约定是调用者保存，它们保存在该进程的内核堆栈中。`ESP` 也不需要保存，因为 context 本身就是堆栈栈顶位置，而进程 PCB 中 `proc->context` 成员就指向 context 就是堆栈对应的位置。于是 `xv6` 中的进程内核断点执行现场 context 只有成员 `edi`、`esi`、`ebx`、`ebp`、`eip`（参见 [proc.h#L16](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L16)）。 

例如，某进程用户态代码执行 `yield()` 系统调用而希望让出 CPU，本进程的内核堆栈中形成的 `trapfeame`。然后随着内核代码的执行和函数调用的嵌套，形成更多层次的函数调用栈，直至 `swtch()` 为止，最后在切换前在堆栈保存现场 context。

### 1.4 proc.h

`proc.h` 文件中定义了 `cpu` 结构体，用于描述个处理器核；定义了全局变量 `cpus[NCPU]` 记录了全部处理器核的信息，以及全局变量 `ncpu` 用于记录处理器核数；声明了外部定义的两个 CPU 私有变量，`cpu` 用户获取自己所在的 CPU 核的指针，`proc` 变量用于获取当前正在运行的进程指 针；定义了 context 用于结构体，用于记录内核切换断点的现场；用于表示进程调度状态的枚举值 `procstate`；最后是描述 `xv6` 进程 PCB 的 `proc` 结构体。

### 1.5 proc.c

`proc.c` 是 `xv6` 进程管理的核心代码。包括一些全局性的变量，例如 `ptable` 用于记录管理所 有进程，其中 `ptable.proc[NPROC]` 数组用于记录所有进程的 PCB；`initproc` 是 `init` 进程的 PCB。

## 2. 进程控制

`xv6` 中使用 `ptable` 中的静态数组 `ptable.proc[]` 来记录和组织进程，请见 [proc.c#L10](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L10)。该数组最大可以记录 `NPROC`（64，参见 [param.h#L1](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L1)）个数的进程，并且由 `lock` 自旋锁保护。该自旋锁通过 [proc.c#L23](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L23) 的 `pint()` 完成初始化（也是 `pint()` 的唯一用处）。 第一个进程的 PCB 由 [proc.c#L15](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L15) 的 `initproc` 指针变量指出。全局变量 `nextpid` 是下一个可用的进程号，并初始化为 1。（貌似 `xv6` 对进程号不重复使用，单项递增，可能考虑到整数范围对 `xv6` 实验而言有足够大）

### 2.1 进程创建与撤销

创建进程时需要分配一个空闲的 PCB，该工作由 [proc.c#L68](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L68) 的 `allocproc()` 完成。[proc.c#L81](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L81) 在进程 PCB 数组 `ptable.proc[]` 中查找未使用的一项。

如果找到空闲 PCB，则 [proc.c#L89](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L89) 将其状态修改为胚胎态 EMBRO，将其进程号 `pid` 设置为 `nextpid`（同时 `nextpid` 自增 1）。注意，进程号 `pid` 和 PCB 在数组 `ptable.proc[n]` 中的位置（数组元素索引号 n）没有必然联系。然后 [proc.c#L94](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L94) 通过 `kalloc()` 分配内核栈空间，`p->kstack` 记录堆栈内存区间的起始位置（低端地址），再进一步将内核陷阱帧 `p->tf` 和堆栈指针 `sp` 指向堆栈内存空间的最高地址减去 `trapframe` 的空间。

然后在堆栈中分配 4 字节的 `trapret` 和 `context` 结构体，这样伪造成一个进程从用户态进入内核态的 `trapframe`，然后在伪造出进程切换断点在 context 中，这样就可以通过进程切换 “返回” 到进程应该执行的位置（例如可能是程序第一个指令，或者 `fork` 之后的下一条指令）。 

早期的 Linux 是将 PCB（`task_struct` 结构体）和内核堆栈放到一起，找到了进程控制块后根据固定偏移量也就找到了内核堆栈。后来的 Linux 堆栈也是和 PCB 分离的，类似于 `xv6` 这种 方式。

### 2.2 第一个进程 initcode

