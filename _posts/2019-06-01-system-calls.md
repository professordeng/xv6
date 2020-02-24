---
title: 1. 系统调用
---

操作系统的功能是替用户进程管理资源，用户进程会要求获得运行、要求获得内存、要求打开文件等。这些代码本质上是用户想执行的操作，但是由于涉及系统资源而归入到内核中，并通过系统调用而得以执行，代表着用户进程在内核态执行。

这部分代码是 “用户进程” 调用的，代表着用户进程的行为（实际上由内核执行）的代码，构成了 XV6 中的所有系统调用的各种 `sys_XXX` 代码。而 XV6 操作系统的 ”主动代码“ 主要就是各种初始化代码（一次性的执行）和周期性执行的 `scheduler()`。而 Linux 设计的要复杂得多（还有页面清洗与回收、磁盘 IO 刷出等操作）。

注意：无论是使用老式的 PIC 还是新的 APIC 作为中断控制器，都不影响 CPU 内部对中断响应的处理过程。 

## 1. 系统调用 / 中断

在 kernel 的 `main()` 中将执行 [tvint()](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L17) 进行中断向量表的初始化。

X86 的 IDT 表共有 256 项，有部分项是与确定事件（除 0、非法指令等异常）相关联的， 它们占据了 0~31 项，剩余的表项可以自由使用。XV6 将 32~63 共 32 个表项与 32 个硬件外设中断相关联，另一些与系统调用的 `INT` 指令相关联。

IDT 表的内容是由 [vectors.pl](https://github.com/professordeng/xv6-expansion/blob/master/vectors.pl) 生成的 `vectors.S` 定义的。可以看到不同的中断最后对转入到公共入口的 [alltraps](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L4)。不同中断将压入不同的中断号 [trapno](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L170) 以便相互区分，因此在执行 `alltraps` 之前堆栈中有相应的中断号。

`alltraps` 作为公共入口继续会调用 [trap()](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L35)，而 `trap()` 将会根据中断号而做不同的分发处理，也许转入 [syscall()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L131) 完成系统调用，或者是个硬件中断服务程序（例如键盘中断对应的 [kbdintr()](https://github.com/professordeng/xv6-expansion/blob/master/kbd.c#L46)）。 

与 X86 上的 Linux 实现类似，XV6 每个系统调用在 IDT 表中占一项，而 Linux 所有系统共用一个入口 `0x80` 项，XV6 和 Linux 都通过一个 IDT 表项的跳转后再经过公共处理代码区分不同功能的系统调用。其实 XV6 系统调用数量较少，也可以考虑每个系统调用占用一个 IDT 表。 但 Linux 有一两百个系统调用，如果每一个占 IDT 中的一项可能会不够用，也无法扩展到超过 256 个以上的系统调用。 

IDT 表仅用于进入中断服务程序，而从中断返回的 `iret` 指令则不依赖于 IDT 表，而是依赖于内核栈中的数据。 

---

从 [traps.h](https://github.com/professordeng/xv6-expansion/blob/master/traps.h) 可以看到前 32 个中断向量号用于内部异常事件，外设 IO 中断使用 [T_IRQ0](https://github.com/professordeng/xv6-expansion/blob/master/traps.h#L30) 以后的向量号。 

[vectors.pl](https://github.com/professordeng/xv6-expansion/blob/master/vectors.pl) 生成的 `perl` 程序是用来生成 `vectors.S` 文件的。 

`vectors.S` 的内容分成两部分：

1. 前面是 256 个代码片段，每个代码片段有一个 `vector0/1/2…/255` 行号记录其起始地址，代码内容则是简单地将对应的编号压入堆栈（内核栈）后跳转到公共入 口 `alltraps` 去处理。
2. 其次是一个地址列表（前面的 `vector0/1/2…/255` 对应的地址）所构成的 IDT 表。

其他 C 语言代码在引用该表时使用的是 `vectors[]` 变量，例如 [trap.c#L13](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L13) 声明为外部变量 `extern uint vectors[];`，链接时将匹配本文件 `vectors.S` 中的 `vectors` 符号。

XV6 在系统启动之后，在 [main.c#L30](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L30) 执行 [tvinit()](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L17) 完成 IDT 表的初始化。

## 2. 系统调用入口

既然系统调用将执行内核代码，因此和普通函数的调用必不相同。应用程序调用 XV6 系统调用时使用的是 `usys.S` 汇编程序中定义的调用代码。[usys.S](https://github.com/professordeng/xv6-expansion/blob/master/usys.S) 定义了 XV6 所提供的全部系统调用，每个系统调用的入口代码都用 `SYSCALL` 宏来声明，应用程序中可以直接用类似 `fork()`、`wait()` 的形式发出系统调用。

我们用 `gcc -E usys.S` 进行预处理，展开所有的宏定义。可以看出系统调用是通过 `int $64` 指令发出的。

```assembly
# 1 "usys.S"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "usys.S"
# 1 "syscall.h" 1
# 2 "usys.S" 2
# 1 "traps.h" 1
# 3 "usys.S" 2
# 11 "usys.S"
.globl fork; fork: movl $1, %eax; int $64; ret
.globl exit; exit: movl $2, %eax; int $64; ret
.globl wait; wait: movl $3, %eax; int $64; ret
.globl pipe; pipe: movl $4, %eax; int $64; ret
.globl read; read: movl $5, %eax; int $64; ret
.globl write; write: movl $16, %eax; int $64; ret
.globl close; close: movl $21, %eax; int $64; ret
.globl kill; kill: movl $6, %eax; int $64; ret
.globl exec; exec: movl $7, %eax; int $64; ret
.globl open; open: movl $15, %eax; int $64; ret
.globl mknod; mknod: movl $17, %eax; int $64; ret
.globl unlink; unlink: movl $18, %eax; int $64; ret
.globl fstat; fstat: movl $8, %eax; int $64; ret
.globl link; link: movl $19, %eax; int $64; ret
.globl mkdir; mkdir: movl $20, %eax; int $64; ret
.globl chdir; chdir: movl $9, %eax; int $64; ret
.globl dup; dup: movl $10, %eax; int $64; ret
.globl getpid; getpid: movl $11, %eax; int $64; ret
.globl sbrk; sbrk: movl $12, %eax; int $64; ret
.globl sleep; sleep: movl $13, %eax; int $64; ret
.globl uptime; uptime: movl $14, %eax; int $64; ret
```

用 `objdump -d usys.o` 得到的结果和刚才分析的一致，可以看到各个系统调用的入口如 `<fork>`、`<exit>` 等等。

```assembly
usys.o:     file format elf32-i386


Disassembly of section .text:

00000000 <fork>:
   0:	b8 01 00 00 00       	mov    $0x1,%eax
   5:	cd 40                	int    $0x40
   7:	c3                   	ret    
...
...
...
00000098 <sleep>:
  98:	b8 0d 00 00 00       	mov    $0xd,%eax
  9d:	cd 40                	int    $0x40
  9f:	c3                   	ret    

000000a0 <uptime>:
  a0:	b8 0e 00 00 00       	mov    $0xe,%eax
  a5:	cd 40                	int    $0x40
  a7:	c3                   	ret    
```

读者需要理清以下细节（我们以 `sleep` 系统调用为例）：用户态代码是通过类似于 `sleep()` 等形式以函数调用的方式进行调用，参数压入本进程用户态堆栈，然后用 `call` 指令跳转到全局符号 `sleep()` 对应的地址处，执行上述信息 `sleep` 对应的三条指令，其中 `int $64` 将引发软中断再次根据中断向量表转向到内核代码处运行，并伴随从用户态堆栈切换到内核堆栈。此时，貌似 `sleep()` 的参数已经无法直接从堆栈找到了，因为堆栈指针已经指向内核栈。但是不要紧，我们仍可以找回用户态堆栈从而可以获得调用参数。

### 2.1 IDT 表初始化

在 kernel 执行 `main()->tvinit()` 时完成设置中断向量表，用 [SETGATE()](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L160) 宏将 `vectors.S` 中定义的 `vector[]` 填写到 0~255 号中断向量。并用 `SETGATE()` 对系统调用 `T_SYSCALL` 单独处理，使得用户态代码（优先级 3）可以通过 `INT` 指令穿过系统调用门。对于中断使用的中断门，而对系统调用使用的是陷阱门 （不关中断）。 

但是直到各处理器启动过程中执行到 `mpamin()->idtinit()` 的时候，才会将 IDT 表装入到 IDTR 寄存器而生效。

然后可以通过外部设备事件、CPU 内部事件、 INT 指令或 CALL 指令通过 IDT 表而产生中断的响应处理过程。

### 2.2 公共入口

不同原因引起的中断经过 IDT 表跳转到 `vector0/1/2…255` 中某一个代码， 这些代码简单地压入中断号之后就跳转到 `alltraps` 公共处理代码处。

[trapasm.S#L6](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L6) 将完成 Trap Frame 的构建。其 中 `pushal` 将压入 8 个寄存器的值到堆栈中。[trapasm.S#L13](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L13) 将段寄存器 DS、ES 设置为 `SEG_KDATA`。

[trap()](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L35) 函数首先判断 `tf->trapno` 是否为系统调用 `T_SYSCALL`，如果是则转到 [syscall()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L131)。

如果中断是硬件引起的，则调用相应的硬件中断处理函数。否则，XV6 认为是发生了错误的操作（例如除以 0 的操作），此时如果是用户态代码引起的则将当前进程 `cp->killed` 标志置位，否则是内核态代码只能 `panic()` 给出警告提示。 

### 2.3 系统调用

发生中断后如果确定是系统调用，则会执行 [syscall()](https://github.com/professordeng/xv6-expansion/blob/master/syscall.c#L131)。根据调用号（`proc->tf->eax`），从 `syscalls[]` 数组对应位置取出函数地址，并跳转执行。  

编号为 n 的系统调用的处理函数 `syscalls[n]`， 还需要获得系统调用参数，这涉及到 `argint()`、`argptr()` 和 `argstr()` 等函数。这些函数也定义在 `syscall.c` 中，供具体的系统调用代码用以获取调用参数。XV6 使用 `trapframe` 里的 `esp` 来确定参数，`argint()` 进一步调用 `fetchint()` 从用户态堆栈读入参数并写入到 `*ip` 里。内核还需要确保指针在进程空间内部，虽然非法地址会引起进程的撤销，但是内核可以访问该进程以外的地址（进程传递了非法的地址指针）。

`syscall()` 返回到 `trap()`，进而返回到 `alltraps` 代码段，最后通过 `iret` 完成系统调用的返回。

### 2.4 trapasm.S

中断的公共入口代码 [alltraps](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L4) 和公共返回代码 [trapret](https://github.com/professordeng/xv6-expansion/blob/master/trapasm.S#L24) 都在 `trapasm.S` 中以汇编形式实现。 需要注意返回代码 `trapret` 的最后一个操作时 `iret`，如果 `iret` 返回到另一个特权级别（将堆栈中返回地址 CS 的 RPL 和当前的 CPL 比较而判定），则在恢复程序执行之前 `iret` 指令还从堆栈弹出堆栈指针 ESP 与堆栈段寄存器 SS。

### 2.5 trap.c

[tvinit()](https://github.com/professordeng/xv6-expansion/blob/master/trap.c#L17) 用 `vectors[]` 中的中断服务程序入口地址加上访问权限形成 “门” 的结构，填写到 IDT 表（`idt[]`）里面。除了其中对应系统调用的 `idt[T_SYSCALL]` 项填写的访问权限是 `DPL_USER`，表明可以从用户态通过该门，其他的 IDT 表的门都不能在用户态调用。其中 `vectors[]` 数组定义在生成的 `vectors.S` 末尾，实际上是标号地址 `vector0~vector255`。而真正将 IDT 装入到 IDTR 要等待 `mpamin()->idtinit()` 的时候。

对于每个外设的中断，在调用相应的处理函数（例如磁盘的 `ideintr()`）之后，还需要执行 `lapiceoi()` 向 `lapic` 芯片通知中断处理已经完成。

### 2.6 syscall.h

该文件定义了 XV6 系统调用的编号，一共有 21 个。

## 3. 数据传递

应用程序直接按照的 C 函数的形式进行系统调用，因此所有的参数都按照函数的参数传递方式，存放在函数的栈帧结构中。但是系统调用中，我们就需要从栈帧中定位参数的位置， 并设法读取相应的参数值。`argint()` 将系统调用的第 n 个参数作为整数指针返回，它从 `trapframe` 中定位第 n 个参数的位置，然后通过 `fetchint()` 检查地址范围并完成参数的值的读取。

### 3.1 返回结果

保存在 `eax` 中。

使用 [copyout()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L362) 拷贝数据到用户进程，`copyout()` 用 `ua2ka()` 将用户地址映射到内核地址。

## 4. 系统调用接口

有关进程管理的系统调用基本上都在 [sysproc.c](https://github.com/professordeng/xv6-expansion/blob/master/sysproc.c)，只是简单地用 `sys_XXX()` 形式的函数封装了一下各种具体系统服务函数，或者就直接实现简单功能的系统调用服务。 

### 4.1 usys.S

[usys.S](https://github.com/professordeng/xv6-expansion/blob/master/usys.S) 给出了用户态可以执行的、对应所有系统调用的函数，其作用相当于 C 语言库对系统调用的封装。例如 `SYSCALL(fork)` 则定义了可供用户调用的 `fork()` 函数，应用程序执行 `fork()` 将会引发 `sys_fork()` 系统调用。编写应用程序的时候，需要借助这些函数来进行系统调用。

