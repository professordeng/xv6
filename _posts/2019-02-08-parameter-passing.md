---
title: 2. 中断机制的参数传递
date: 2019-07-03
---

操作系统在开始运行用户进程的时候，内核便开始处于被动状态，只有在出现以下几种情况的时候才会触发硬件机制陷入内核：

1. 用户代码由于某种原因引发异常（例如除以 0）。
2. 硬件产生中断并且没有屏蔽触发中断。
3. 用户代码调用相关指令（例如 `x86` 体系下的 `int` 系统调用指令）主动陷入内核。

以上三种情况便是异常、中断、系统调用机制。这三种机制由于需要陷入内核所以在进入内核之前必须先保存现场，然后回到用户环境的时候恢复现场，`xv6` 对着三种机制都采用相同的处理方式陷入内核。

## 1. 保护现场

在 `x86` 体系下，这三种机制都会触发相同的硬件操作，完成部分保护现场的任务，通过将部分寄存器值压入堆栈来实现保护现场，如果触发中断（由于系统调用、中断、异常具有相同的处理机制，所以一下全以中断代称）前处于内核态，则直接在当前栈中保护现场，如果处于用户态，则根据任务栈描述符得到新的内核栈并压入用户态的 `ss` 和 `esp`。

`x86` 规定了中断、异常、系统调用的规范，在发生以上情况时，硬件能够区分上述规定的 256 种触发中断的原因并通过寻找在内存中保存的中断向量表来得到中断处理程序的地址，然后将控制权交给中断处理程序。

`xv6` 通过一个存放函数指针的数组来作为中断向量表，在 `main()` 函数初始化过程中，调用 `tvinit()` 函数完成中断向量表的初始化。`tvinit()` 将每一个中断处理程序的地址写入 `idt()` 数组中，`idtinit()` 将数组地址载入中断向量表寄存器，硬件能够根据中断向量表寄存器准确找出中断处理程序，`xv6` 使用 `vector.pl` 脚本生成 `vector` 数组，`vector` 数组存放着每个中断处理程序的入口地址，`xv6` 简单地将所有的中断处理程序指向 `alltraps`，由 `alltraps` 来负责具体的处理。在调用 `alltraps` 之前，`xv6` 统一压入 `errnum` 和 `trapnum` 来区分是 256 情况中的哪种。

`alltraps()` 继续压入寄存器保存现场，得到 `trapframe` 结构体。

在这之后重新设置段寄存器，进入内核态，压入当前栈 `esp`，然后调用 C 函数 `trap()` 处理中断，在 `trap()` 返回时，弹出 `esp` 然后 `trapret()` 弹出寄存器恢复现场。

在这里有必要详细说明的是在调用 `trap()` 时由于 `trap()` 是 C 函数所以会在栈上压入返回地址 `eip` 和部分寄存值构成 context 结构，context 结构紧接着 `trapframe` 处于内核栈中。

之所以会重复压入部分寄存器的值是因为 Intel 规定 `esi`，`ebx`，`esi` 是被调用者保存寄存器，需要由被调用者保存，而 `ebp` 则是 C 函数中维护每个函数栈帧用的，`eip` 是 call 指令压栈的，也就是说，当 call 指令执行后会执行以上动作才跳入 `trap()`，具体有关被调用者寄存器可以参考下一篇文章。

在 `trap()` 执行 `ret` 指令返回之前会弹出 context 恢复栈的调用前状态。

明确 context 结构能够使我们很容易理解进程调用时，通过切换下文 context，很容易便能实现切换进程的目的，因为所有进程在被调度前都会运行 `trap()`，在让出 CPU 之前便已经把现场保存好了。

实际调度的情况是这样的，当把当前进程上下文切换至另一个 context 上下文时，调度器会手动弹出 context 结构来模拟 `trap()` 返回的情形，当弹出 `eip` 的时候就意味着回到了 `call trap` 指令的下一条指令中来。而此时 `trapframe` 中的返回地址是另一个进程的用户代码，这样便能实现进程的切换。

## 2. 第一次运行进程

进程调度器通过恢复进程现场来返回进程，此时并没有所谓的进程 “现场”，那么进程又是如何第一次运行的呢？

`xv6` 通过自建一个 “现场” 来模拟返回第一个进程，`xv6` 手动写入了 `trapframe` 和 context 结构的上下文来让调度器调度并返回用户进程，这些操作由 `allocproc` 负责，`allocproc` 手动构建内核栈并写入内核寄存器来构成进程第一次运行的 “现场”，代码参见 [proc.c#L94](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L94)。

`allocproc` 在 “正常现场” 中本应该压入 `esp` 的地方压入 `trapret`，`trapret` 处的代码负责弹出寄存器恢复现场，然后在本应该是 `trap()` 返回地址的地方压入 `forkret` 的地址并将 context 的内容置零，通过这种方式，调度器在调度的时候手动弹出 context，但是此时弹出的 `eip` 并不是 `trap()` 的返回地址而是 `forkret`，`forkret` 返回时弹出压入的 `trapret` 恢复现场，这样进程第一次便开始运行了。

## 3. C 中断处理程序 trap

`trap()` 主要根据 `trapframe` 中的 `trapno` 来确定到底是哪种原因导致中断的发生。

### 3.1 系统调用

如果是系统调用，则通过调用 `syscall()` 函数负责具体的系统调用处理，参见 [trap.c#L39](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L39)。

[trap.c#L42](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L42) 重新更新了 `tf` 的地址，这样做的原因是 `trapframe` 的大小和 `ss` 和 `esp` 有关，而任务栈 `ts` 总是指向内核栈所在页的最高地址处：

单独处理 [proc->killed](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L44) 的原因是 `kill` 系统调用的需要，`kill` 系统调用通过将 `killed` 置 1，来杀死一个进程，由于迟早进程会由于系统调用或者时钟中断进入 `trap()`，此时 `trap()` 检查到 `killed` 置 1 便能够将进程杀死。

### 3.2 硬件中断或异常

如果中断产生的原因是硬件中断或者异常，`trap()` 则调用相应的函数来进行处理，参见 [trap.c#L49](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L49)。

### 3.3 系统调用实现机制

那么如果是系统调用，`syscall()` 都干了些什么事情，这里只是讲讲系统调用的实现机制。

`syscall()` 通过 `trapframe` 中的 `eax` 来确定系统调用号以决定调用那个系统函数，当然 `eax` 的值或许是库函数在调用 `int` 指令的时候设置的，只是保护现场使得存放在了 `trapframe` 中，然后通过系统调用号调用具体的系统调用处理函数并返回到 `trapframe` 中的 `eax` 位置，这样恢复现场时库函数便能根据 `eax` 得到系统调用的返回值。

`xv6` 具体的系统调用参见 [syscall.c#L107](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L107)。每个系统调用都有不同的参数，那么在内核中的系统调用函数又是如何找到这些参数？参数或许是库函数在陷入内核前压栈的，所以可以根据 `trapframe` 中的用户栈 `esp` 来找到各种参数，`xv6` 使用了工具函数 [argint()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L48)、[argptr()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L55) 和 [argstr()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L72) 来获得第 n 个系统调用参数。三个函数功能如下

他们分别用于获取整数，指针和字符串起始地址。`argint()` 利用用户空间的 `%esp` 寄存器定位第 n 个参数：`%esp` 指向系统调用结束后的返回地址。参数就恰好在 `%esp` 之上（`%esp+4`）。因此第 n 个参数就在 `%esp+4+4*n`。

`argint()` 调用 [fetchint()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L16) 从用户内存地址读取值到 `*ip`。`fetchint()` 可以简单地将这个地址直接转换成一个指针，因为用户和内核共享同一个页表，但是内核必须检验这个指针的确指向的是用户内存空间的一部分。内核已经设置好了页表来保证本进程无法访问它的私有地址以外的内存：如果一个用户尝试读或者写高于（包含）`p->sz` 的地址，处理器会产生一个段中断，这个中断会杀死此进程，正如我们之前所见。但是现在，我们在内核态中执行，用户提供的任何地址都是有权访问的，因此必须要检查这个地址是在 `p->sz` 之下的。

`argptr()` 和 `argint()` 的目标是相似的：它解析第 n 个系统调用参数。`argptr()` 调用 `argint()` 来把第 n 个参数当做是整数来获取，然后把这个整数看做指针，检查它的确指向的是用户地址空间。注意 `argptr()` 的源码中有两次检查。首先，用户的栈指针在获取参数的时候被检查。然后这个获取到的参数作为用户指针又经过了一次检查。

`argstr()` 是最后一个用于获取系统调用参数的函数。它将第 n 个系统调用参数解析为指针。它确保这个指针是一个 NULL 结尾的字符串并且整个完整的字符串都在用户地址空间中。

系统调用的实现（例如，`sysproc.c` 和 `sysfile.c`）仅仅是封装而已：他们用 `argint()`，`argptr()` 和 `argstr()` 来解析参数，然后调用真正的实现。`sys_exec` 利用这些函数来获取参数。