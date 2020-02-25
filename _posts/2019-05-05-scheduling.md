---
title: 5. 进程调度
---

进程调度主要涉及两个方面：一个是调度器，一个是进程切换代码。XV6 的进程调度器比较简单，只是简单扫描就绪进程并选择下一个可运行的进程；而切换代码则略显复杂，涉及切换现场的保存和恢复，而且和任务状态段 TSS 的细节有关。 

## 1. 进程调度

### 1.1 scheduler

每个 CPU 在启动后将调用 [scheduler()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L314)，该函数是一个无限循环，不断选择下一个就绪进程并且换过去执行。在主 CPU 启动过程中 [main()->mpmain()](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L57)，以及其它处理器的 [mpenter()->mpmain()](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L47) 中开始启动。 

`scheduler()` 代码的安排如下：外层是一个无限循环；内层循环将扫描遍历 `ptable->proc[]` 数组，如果找到一个 [RUNNABLE](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L339) 的就绪进程，则切换过去执行。其中指针 [c](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L326) 指向 “每 CPU 变量”，指针 `p` 记录当前 CPU 上正在执行的进程。可以看出其调度算法实际上就是轮转算法，按照进程在 PCB 表 `ptable->proc[]` 中的存放次序轮转（而不是根据进程号轮转）。

进程切换过程则通过 [switchuvm()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L343) 完成进程 TSS 切换和用户空间的切换，使得切入进程影像可见。然后设置切入进程状态 `p->state=RUNNALBE`。`swtch()` 跳出内核运行现场，恢复切入进程的切换断点的现场，从此执行被调度的进程。

注意：只有到被调度进程因各种原因让出 CPU 而执行 [sched()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L358) 的时候，才会从 `swtch()`返回。一旦返回，则 `scheduler()` 将要重新接管 CPU，通过 `switchkvm()` 将页表恢复成 `scheduler()` 自己的页表（仅在内核空间有映射），最后将 `c->proc=0` 表示当前 CPU 没有运行用户进程， 然后进入下一次调度循环选取下一个就绪进程。

由于 `scheduler()` 是在内核态运行的，因此用户空间的切换 `switchuvm()` 执行过后，对 `scheduler()` 本身的运行没有任何影响，运行的仍是内核的 `scheduler()` 代码。

### 1.2 sched

[sched()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L358) 函数是用户进程切换到 scheduler 执行流时所调用的代码，scheduler 切换到用户进程是在 `scheduler()` 调用 `swtch()` 完成的。它在进程主动睡眠、时间片轮转、IO 读写需要阻塞、主动调用 `yield()` 放弃 CPU 等情况下都会被调用。`sched()` 的作用是从当前进程切换到 scheduler 执行流，从而让出 CPU。 

需要注意 `sched()` 调用的条件，首先必须持有 `ptable.lock` 锁。

当前进程不能仍处于 `RUNNING` 状态，按理应该处于 `RUNNABLE`、`SLEEPING`、`ZOMBIE` 状态。例如

1.  执行 `sleep()` 睡眠操作时，必须先将当前进程状态修改为 [SLEEPING](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L440)，然后再调用 `sched()`。
2.  执行 `exit()` 退出操作时，需要先将当前进程状态修改为 [ZOMBIE](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L265) 后再调用 `sched()`。
3.  执行 `yield()` 让出 CPU 时，当前运行进程的状态修改为 [RUNNABLE](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L389)，再调用 `sched()`。

`sched()` 和 `scheduler()` 的切换过程在 “切换断点” 的处理上是相同的，都是通过 `swtch()` 完成 context 现场的切换。 

### 1.3 切换时机

XV6 执行的是时间片轮转算法，查看 [trap()](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L103) 你会发现，如果进程号不是 0 并且进程的状态是 RUNNING ，中断条件为时间片轮转引起的中断号，就调用 [yield()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L384) 放弃 CPU，然后进入调度流重新调度。

## 2. 执行流的切换

这里的 “执行流切换” 指的是：

1. 从 scheduler 的执行流切换到切入进程的执行流的过程。
2. 从换出进程的执行流切入到 scheduler 的执行流。

执行流的切换也将伴随着空间的切换，即页表的切换。 

进程执行流切换时，正在执行的待换出进程是处于运行 scheduler 状态，也就是说页表是 scheduler 的页表。所以换出和切入的两个进程是在内核态执行现场的切换。因此，读者需要完整理解这个过程，就首先需要了解从用户态进入到内核态的系统调用和中断过程。

### 2.1 进程影像的切换

为了从 scheduler 执行流切换到切入进程，需要用 [switchuvm()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L155) 完成进程影像切换，此时 scheduler 执行流使用的是切入进程的影像（段表和页表）。切换段表的原因是每个进程有自己私有的 `SEG_KDATA`（用于保存 CPU 私有变量）以及 TSS 中保存的切入进程内核栈起点指针。 然后经过 `swtch()` 切换到切入进程执行流。 

从换出进程返回到 scheduler 执行流的时候，则通过 [switchkvm()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L147) 完成进程影像切换到调度器的内存影像，仅需切换页表，因为 scheduler 执行流仍可以访问 CPU 私有变量 `cpu` 获得当前进程 PCB，而且 scheduler 无需使用（切换） TSS 中的信息，所以仍可以使用该进程的段表和页表。

### 2.2 swtch

[swtch()](https://github.com/professordeng/xv6-expansion/blob/master/swtch.S) 是在内存影像已经完成切换之后才执行的，负责：

1. CPU 运行现场 context 切换。
2. 内核堆栈切换。
3. 指令 EIP 的跳转 。

`swtch()` 虽然是汇编代码片段，但是它相当于 C 语言函数 `void swtch(struct context *old, struct context *new)`，其参数按照 C 函数规范在堆栈中指定的位置。

这里切换代码中没有出现 context 中的 `eip` 成员，它是硬件操作的，通过函数调用的 call 指令压入 `eip` 堆栈形成 context 的 `eip` 成员，同理对于新的 context 在执行 `ret` 时会从堆栈中弹出 `eip`。  

通过 `swtch()` 完成切换断点的现场切换，用切入进程的切换断点现场替代换出进程的切换断点现场。[swtch.S#L20](https://github.com/professordeng/xv6-expansion/blob/master/swtch.S#L20) 进行堆栈切换，也完成了 context 的切换，因为此时的 `esp` 指向的就是 context 内容。 

下面我们来分析 `swtch()` 的 `ret` 执行前后是如何完成执行流的切换的。首先要认识到的是，被换出进程执行流的断点是在 [sched()->swtch()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L380) 的下一条语句。因此一个进程在执行 `sched()` 的过程中，不是一次性完成的，在断点处将会切换到 `scheduler()` 执行流，而恢复运行的时候则是从 `scheduler()` 执行流返回到这个断点 [mycpu()->intena=intena;](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L381) 继续运行。其次要认识到 `scheduler()` 的执行流断点是在 `scheduler()->swtch()` 的一下条语句，也就它的断点 EIP 指向的是 [switchkvm();](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L347)。因此 `scheduler()` 函数也不是一次性完成的，在断点处将会切换到调度的就绪进程，而恢复运行时则从那个进程的执行流返回到断点 `switchkvm()` 继续运行。

### 2.3 返回到用户态

进程在内核态完成切换，最终还要返回到用户态断点，才能恢复用户进程继续运行。我们还是借助于内核栈的 `trapframe`，下面以 B 进程的堆栈为例。当内核代码逐层返回后最终将遇到 `trapframe` 帧，而无论是系统调用还是中断返回，都会执行 [trapret](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L23) 代码。也就是说 B 进程在内核态经过 `sleep()` 函数返回，最终执行到 `trapret` 代码，它将从内核堆栈恢复现场：`popal` 弹出通用寄存器、然后弹出并恢复原来的 GS、FS、ES、DS，弹出 `trapno` 和 `errcode`，最后执行 `iret`，此时将返回到 `trapframe`，一开始就记录的用户态断点（CS、EIP 和 EFLAGS），返回到用户态是从 `ring0` 转到 `ring3`，因此会从堆栈 `trapframe` 中恢复用户态堆栈 SS 和 ESP（当用户态经系统调用 / 中断进入内核态，发生从 `ring3` 转到 `ring0` 的运行级提升时，堆栈要切换到内核栈是借助 TSS 中的 `SS0` 的 `ESP0` 完成的（由硬件自动完成）。 

### 2.4 进程调度时堆栈切换问题

进程放弃在 CPU 上运行的时候，调用 [swtch()](https://github.com/professordeng/xv6-expansion/blob/master/swtch.S#L3) 来保存自己的切换断点执行环境（struct context），并且换到 scheduler 的执行环境。

读者需要将中断的进入和退出操作，与进程运行环境切换操作的进行关联，才能构成内核栈的全景认识。

scheduler 的执行环境 context 记录在 [cpu->scheduler](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L4) 上，因此任何一个进程都可以通过 `cpu->scheduler` 找到对应的执行环境完成第一步切换步骤。而进入到 scheduler 之后，将会扫描进程列表 `ptable->proc[]` 找到就绪进程，并再通过一次 `swtch()` 完成执行环境的切换。