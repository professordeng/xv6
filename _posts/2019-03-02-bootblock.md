---
title: 2. bootblock（调试）
---

从这里开始分析 XV6 的启动过程。启动的整个流程包括 `BIOS` → `bootloader` → `kernel`，XV6 的 `bootloader` 是启动扇区 `bootblock` 文件，`kernel` 是内核文件 `kernel`。XV6 的启动包括主 CPU 启动和其他 CPU 启动两个分支，其中主 CPU 的启动过程才是重点。 

1. `bootblock` 启动的相关代码

   [bootasm.S](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S)、[bootmain.c](https://github.com/professordeng/xv6-expansion/blob/master/bootmain.c)

2. `kernel` 启动的相关代码

   [entry.S](https://github.com/professordeng/xv6-expansion/blob/master/entry.S)、[main.c](https://github.com/professordeng/xv6-expansion/blob/master/main.c)、[scheduler()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L314)

3. 其他 CPU 启动的相关代码

   [entryother.S](https://github.com/professordeng/xv6-expansion/blob/master/entryother.S)、`main.c`、`scheduler()`

在 `entry.S` 中启动分页时采用了 4 MB 大页模式，简化编程；而到了 `main()->kvmalloc()` 之后则启用了 4 KB 的页，以提高物理页帧的利用率。

每个处理器启动完成后，都将进入 `scheduler()` 内核执行流，不断查找下一个就绪的进程并切换过去执行，进入调度循环。 

## 1. bootblock

在当前 PC 上要运行 X86 实模式的程序的有两种：一种是运行 DOS 系统；另一种是引导扇区的代码，在系统启动时还没进入保护模式前获得运行。`bootblock` 则属于上述第二种，需要运行 16 bit 代码的情况。

但是，直到 1995 年以后，GNU 汇编器 AS 才逐步加入编写 16 位代码的能力。Linux 刚诞生的时候（1991 年），使用 AS86 汇编器来编写自己的 16 位启动代码。因为 Linux 自从其诞生起就是 32 位，就是多用户多任务操作系统，所以 GNU AS 一移植到 Linux 上就是用来编写 32 位保护模式的代码的。而且 ELF 可执行文件格式也只有 ELF 32 和 ELF 64，没听说过有 ELF 16 的。因此用 GNU AS 来写 16 位汇编并不那么常见，`bootasm.S` 中的 [.code16](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L10) 就是用于指示 GNU AS 生成 16 bit 代码。 

此时我们启动 GDB 对启动代码 `bootblock.o` 进行调试。

1. 终端输入 `make qemu-nox-gdb`
2. 因此命令中的目标是 `bootblock.o` 而不是 `kernel`，需要先在目录下的 `.gdbinit` 下修改 `symbol-file kernel` 为 `symbol-file bootblock.o`，然后在另一个终端输入 `gdb -silent bootblock.o`

我们先用 `l start` 查看第一条指令位置的汇编程序，然后用 `b 13` 将断点设置在第一条 `cli` 指令处，按 `c` 执行 。此时由于是 16 位代码，用 `i r` 命令（即 `info registers`）查看到的寄存器都是 16 位系统上的寄存器。

```bash
eax            0xaa55	43605
ecx            0x0	0
edx            0x80	128
ebx            0x0	0
esp            0x6f04	0x6f04
ebp            0x0	0x0
esi            0x0	0
edi            0x0	0
eip            0x7c00	0x7c00 <start>
eflags         0x202	[ IF ]
cs             0x0	0
ss             0x0	0
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
```

然后我们用 `l start32` 查看 32 位代码的起始处，并用 `b 56` 将断点设置在 32 位代码的第一条汇编语句处，按 `c` 执行，再用 `i r` 命令查看寄存器如下：

```bash
eax            0x11	17
ecx            0x0	0
edx            0x80	128
ebx            0x0	0
esp            0x6f04	0x6f04
ebp            0x0	0x0
esi            0x0	0
edi            0x0	0
eip            0x7c31	0x7c31 <start32>
eflags         0x6	[ PF ]
cs             0x8	8
ss             0x0	0
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
```

## 2. bootasm.S

BIOS 装入 `bootblock` 后（位于 `0x7c00`）将会把控制权转移给 XV6 的 `bootblock`，从此 XV6 的启动代码开始接管系统启动过程。`bootblock` 是由  `bootasm.S` 和 `bootmain.c` 生产的。

`bootblock` 的启动过程分为实模式部分和保护模式部分。

### 2.1 实模式代码

[.code16](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L10) 指出这是 16 bit 代码，可运行于 X86 的实模式。GNU AS 汇编器使用的汇编语言采用的是 AT&T 语法（Linux 环境下的通用标准），该语法和 Intel 语法不同。 

`bootloader` 的第一条指令是 [cli](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L13)  ，用于关闭中断。硬件可以通过中断触发中断处理程序，从而调用操作系统的功能。BIOS 作为一个小型操作系统，为了初始化硬件设备，可能设置了自己的中断处理程序。但是现在 BIOS 已经没有了控制权，而是引导加载器正在运行，所以现在还允许中断不合理也不安全。xv6 会在合适的时机设置 IDT 表然后再打开中断。

在刚进入 `bootblock` 的时候，硬件处于实地址模式，在此模式下有 8 个 16 位的寄存器可用，但实际上处理器发送给内存的是 20 位的地址（[bootasm.S#L21](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L21)）。也就是真正可寻址的范围是 20 位地址（1 M 的地址空间）。当程序用到一个内存地址时，实模式下硬件自动把段寄存器的值（高 4 位）作为 segment 左移四位然后加上该地址的方式来得到线性地址（`segment<<4` + `offset`）。因此，内存引用中其实隐含地使用了段寄存器的值：取指会用到 `%cs`，读写数据会用到 `%ds`，读写栈会用到 `%ss`。

| 虚拟地址 | 线性地址              | 物理地址 |
| -------- | --------------------- | -------- |
| offset   | segment << 4 + offset | 页表查找 |

XV6 假设 X86 指令在做内存操作时使用的是 “虚拟地址”，但实际上 X86 指令使用的是 “逻辑地址”。逻辑地址由段选择器和偏移组成，有时又被写作 `segmemt:offset`。更多时候，段是隐含的，所以程序会直接使用偏移。分段硬件会完成上述处理，从而产生一个 "线性地址"。如果允许分页硬件工作，分页硬件则会把线性地址翻译为物理地址；否则处理器直接把线性地址看作物理地址。

引导加载器还没有允许分页硬件工作；它通过分段硬件把逻辑地址转化为线性地址，然后直接作为物理地址使用。XV6 会配置分段硬件，使之不对逻辑地址做任何改变，直接得到线性地址，所以线性地址和逻辑地址是相等的。由于历史原因我们用 "虚拟地址" 这个术语来指程序操作时用的地址。XV6 的虚拟地址等于 X86 的逻辑地址，同样也等于分段硬件映射的线性地址。等到开启了分页后，系统中值得关心的就只有从线性地址到物理地址的映射。

在 BIOS 中已经将 `%cs` 设为 0 了，但，`%ds`，`%es`，`%ss` 的值是未知的，所以在屏蔽中断后，引导加载器的第一个工作就是将 [%ax](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L15) 置零，然后把这个零值拷贝到三个段寄存器中。

虚拟地址 `segment:offset` 可能产生 21 位物理地址，但 Intel 8088 只能向内存传递20位地址，所以它截断了地址的最高位：0xffff0 + 0xffff = 0x10ffef，但在 8088 上虚拟地址 0xffff:0xffff 则是引用物理地址 0x0ffef。早期的软件依赖硬件来忽略第 21 位地址位，所以当 Intel 研发出使用超过20位物理地址的处理器时，IBM 就想出了一个技巧来保证兼容性。那就是，如果键盘控制器输出端口的第 2 位是低位，则物理地址的第 21 位被清零；否则，第 21 位可以正常使用。引导加载器用 IO 指令控制端口 [0x64](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L24) 和 [0x60](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L37) 上的键盘控制器，使其输出端口的第 2 位为高位，来使第 21 位地址正常工作。

对于使用内存超过 65536 字节的程序而言，实模式的 16 位寄存器和段寄存器就显得非常困窘了，显然更不可能使用超过 1 M 字节的内存。X86 系列处理器在 80286 之后就有了 "保护模式"。保护模式下可以使用更多位的地址，并且（80386之后）有了 “32位” 模式使得寄存器、虚拟地址和大多数的整型运算都从 16 位变成了 32 位。XV6 引导程序依次允许了保护模式和 32 位模式。

XV6 几乎没有使用段；取而代之的是分页机制。引导加载器将段描述符表 [gdt](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L78) 中的每个段的基址都置零，并让所有段都有相同的内存限制（4 G 字节）。该表中有一个空指针表项，一个可执行代码的表项，一个数据的表项。代码段描述符的标志位 [SEG_ASM](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L11) 中指示了代码只能在 32 位模式下执行。正是由于这样的设置，引导加载器在进入保护模式时，逻辑地址才会直接映射为物理地址。

`bootblock` 设置 [GDT](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L39) 为进入到保护模式做准备，而且要保证代码 “无缝” 平滑地执行。此时通过 `lgdt` 指令将 [gdtdesc](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L85) 地址处的数据装入 GDTR 寄存器。其中 GDTR 寄存器包括一个 16 bit（`.word`）表的长度，以及 32 bit（`.long`）的表起始地址。可以看出 GDT 表的起始地址在 [gdt](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L80)，而 `gdt` 地址处有三个 `GDT` 表项。第一个表项使用 `SEG_NULLASM` 宏来生成，而后面两项用 `SEG_ASM` 宏来生成，这两种宏定义于 [asm.h](https://github.com/professordeng/xv6-expansion/blob/master/asm.h)。

1. 第一项用 `SEG_NULLASM` 宏生成的项是全 0
2. 第二项 `SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)` 对应 `bootblock` 代码段，起点为 `0x0`，长度为 `0xffffffff`，类型为 `STA_X|STA_R`（可执行可读）
3. 第三项 `SEG_ASM(STA_W, 0x0, 0xffffffff)` 对应 `bootblock` 数据段，起点为 `0x0`，长度为 `0xffffffff`，类型为 `STA_W`（可写）

设置好 GDTR 后，引导加载器将 [%cr0](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L43) 中的 `CR0_PE` 位置为 1，从而开启保护模式。许保护模式并不会马上改变处理器把逻辑地址翻译成物理地址的过程；只有当某个段寄存器加载了一个新的值，然后处理器通过这个值读取 GDT 的一项从而改变了内部的段设置。

我们没法直接修改 `%cs`，所以使用了一个 [ljmp](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L51) 指令。跳转指令会接着在 [start32](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L54) 执行，但这样做实际上将 `%cs` 指向了 `gdt` 中的一个代码描述符表项。该描述符描述了一个 32 位代码段，这样处理器就切换到了 32 位模式下。就这样，引导加载器让处理器从 8088 进化到 80286，接着进化到了 80386。

### 2.2 保护模式代码

在 32 位模式下，引导加载器首先用 [SEG_KDATA](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L56) 初始化了数据段寄存器。逻辑地址现在是直接映射到物理地址的。运行 C 代码之前的最后一个步骤是在空闲内存中建立一个栈。内存 0xa0000 到 0x100000 属于设备区，而 XV6 内核则是放在 0x100000 处。引导加载器自己是在 0x7c00 到 0x7d00。本质上来讲，内存的其他任何部分都能用来存放栈。引导加载器选择了 0x7c00（即 [$start](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L12)）作为栈顶；栈从此处向下增长，直到 0x0000，不断远离引导加载器代码。

虽然前面的 16 bit 的代码，后面是 32 bit 代码（使用 `.code32` 指示编译器按 32 位编译），但运行起来就如前后两条指令连续运行一般。 

保护模式程序地址是用逻辑地址来表示的，逻辑地址通常保存成 `segment:offset` 的形式。也就是说，此时所有段的起点都是 0，弱化了 X86 的段管理功能。Linux 页采用类似的方式来使用 X86 硬件的内存地址部件。

进入保护模式后，开始设置各个段的索引，可以看到 [DS](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L55)、ES、SS 三个选择字都索引到了数据段 [SEG_KDATA](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L16)，FS 和 GS （GS 段后面会被用作 “每 CPU 变量”，`cpu` 和 `proc` 的特殊段）选择字则指向无效段 [SEG_NULLASM](https://github.com/professordeng/xv6-expansion/blob/master/asm.h#L5)。启动保护模式后的代码段和数据段都映射到 `0~0xffff-ffff` 的线性地址范围。通过这样的设置使得 XV6 可以无视分段机制的地址映射问题，将逻辑地址和线性地址等效地使用。也就是说无论是用 CS 的取指令、DS / ES 的访问数据或用 SS 的访问堆栈，段的起点都是 0，起关键作用的是各自相应的段内偏移地址。 

- 段寄存器设置为 0，32 位地址空间的布局如下

| 地址          | 描述               |
| ------------- | ------------------ |
| `0x0000`      | 起始地址           |
| `0x7c00`      | 启动扇区的起始地址 |
| `0x7d00`      | 启动扇区的结束地址 |
| `0xffff-ffff` | 结束地址           |

然后设置堆栈指针 [%esp](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L64) 为 [$start](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L12)，由于 BIOS 将 `bootblock` 装载到 `0x7c00` 地址处，也就是说 `start` 地址就是 `0x7c00`，由于 `bootblock` 只站 1 个扇区共 512 字节，因此占用地址空间为 `0x7c00~0x7d00`。然后跳到 C 代码执行 `bootmain()`，`bootblock` 的汇编部分至此结束。

最后加载器调用 C 函数 [bootmain](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L66)。`bootmain` 的工作就是加载并运行内核。只有在出错时该函数才会返回，这时它会向端口 [0x8a00](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L70) 输出几个字。在真实硬件中，并没有设备连接到该端口，所以这段代码相当于什么也没有做。如果引导加载器是在 PC 模拟器上运行，那么端口 0x8a00 则会连接到模拟器并把控制权交还给模拟器本身。无论是否使用模拟器，这段代码接下来都会执行 [spin](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L75) 死循环。而一个真正的引导加载器则应该会尝试输出一些调试信息。

例如后面 `bootmain()` 读入磁盘数据后发现不是 ELF 格式（即没有发现有效的内核）。这些代码执行两个 `outw` 指令的 IO 操作，向 `0x8a00` 端口写入数据从而引发 `Bochs` 虚拟机的 `breakpoint`（真实机器在 `0x8a00` 地址只是普通内存没有任何特定作用），然后进入无限循环。 

至此，结束了  `bootasm.S` 中汇编代码的分析，开始第二步骤 C 语言的 `bootmain()` 代码运行阶段。

### 2.3 调试

由于 XV6 的默认设置中，其 GDB 初始化脚本 `.gdbinit` 的最后一行的命令指出所使用的符号表是 `kernel`，因此是无法处理 `bootblock` 的代码和符号的。 

因此我们由两种方法解决：一是将最后的一行从 `symbol-file kernel` 修改为 `symbol-file bootblock.o` 或 `file bootblock.o`；二是直接在 GDB 启动后执行 `file bootblock.o` 命令，替换调试目标程序以及符号表。 

此时再执行 GDB 调试，可以查看 `bootblock` 的信息。我们执行 `l start` 查看 `bootblock` 最初的几条指令，并用 `p/x $cs` 查看到 `cs=0xf000`，用 `p/x $eip` 查看到 `eip=0xfff0`，正处于 PC 还未执行 BIOS 的时候。 

我们用 `b start` 将断点设置在 `bootblock` 的第一条指令，并用 `c` 命令执行到该断点，检查看到 `cs=0x0`、`eip=0x7c00`。此时正是 BIOS 刚跳转到 `bootblock` 的第一条指令而将控制权转交给 `bootblock` 的时刻。 

到保护模式后，GDB 因权限不够而无法读写 `gdt` 等寄存器。此时需要用 `qemu` 的调试功能，在 `qemu` 终端执行 `Ctrl + A`，再执行 `c`，即可进入 `qemu` 的 `monitor` 模式（在此模式下按 `q` 退出），接着输入 `info registers`，可以将整个系统的全部寄存器信息打印出来。

### 2.4 代码回顾

此时我们已经对 `bootasm.S` 的代码已经有了完整的了解。其中定义了一个全局变量 [start](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L11) ，即 XV6 的第一条指令 `cli` 的地址。[start32](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L54) 是开始进入保护模式代码的地址，并通过设置段寄存器构造出 `0~0xffff-ffff` 的线性地址范围。最后通过 [call bootmain](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L66) 跳到 [bootmain()](https://github.com/professordeng/xv6-expansion/blob/master/bootmain.c#L17) 函数。正常情况是不会从 `bootmain()` 返回的，[bootasm.S#L68](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S#L68) 之后的代码对应于异常情况。 

