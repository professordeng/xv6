---
title: 8. bootblock
date: 2019-04-15
---

从这里开始分析 `xv6` 的启动过程。启动的整个流程包括 `BIOS` → `bootloader` → `kernel`，`xv6` 的 `bootloader` 是启动扇区 `bootblock` 文件，`kernel` 是内核文件 `kernel`。`xv6` 的启动包括主 CPU 启动和其他 CPU 启动两个分支，其中主 CPU 的启动过程才是重点。 

1. `bootblock` 代码

   `bootasm.S` 、`bootmain.c`

2. `kernel` 代码

   `entry.S`、`main.c`、`scheduler()`

3. 其他 CPU

   `entryother.S`、`main.c`、`scheduler()`

在 `entry.S` 中启动分页时采用了 4 MB 大页模式，简化编程；而到了 `main()->kvmalloc()` 之后则启用了 4 KB 的页，以提高物理页帧的利用率。

每个处理器启动完成后，都将进入 `scheduler()` 内核执行流，不断查找下一个就绪的进程并切换过去执行，进入调度循环。 

## 1. bootblock

在当前 PC 上要运行 `x86` 实模式的程序的情况有两种：一种是运行 DOS 系统；另一种是引导扇区的代码，在系统启动时还没进入保护模式前获得运行。`bootblock` 则属于上述第二种需要运行 16 bit 代码的情况。

但是，直到 1995 年以后，GNU 汇编器 AS 才逐步加入编写 16 位代码的能力。即便是 Linux 刚诞生的时候（1991 年），也只有使用 `AS86` 汇编器来编写自己的 16 位启动代码。因为 Linux 自从其诞生起就是 32 位，就是多用户多任务操作系统，所以 GNU AS 一移植到 Linux 上就是用来编写 32 位保护模式的代码的。而且 ELF 可执行文件格式也只有 ELF 32 和 ELF 64，没听说过有 ELF 16 的。因此用 GNU AS 来写 16 位汇编并不那么常见，`bootasm.S` 的第 10 行专门指出 `.code16` 就是用于指示 GNU AS 生成 16 bit 代码用的。 

此时我们启动 `gdb` 对启动代码 `bootblock.o` 进行调试。

1. 终端输入 `make qemu-nox-gdb`
2. 因此命令中的目标是 `bootblock.o` 而不是上次的 `kernel`，先在目录下的 `.gdbinit` 下修改 `symbol-file kernel` 为 `symbol-file bootblock.o`，然后在另一个终端输入 `gdb -silent bootblock.o`

我们先用 `l start` 查看第一条指令位置的汇编程序，然后用 `b 13` 将断点设置在第一条 `cli` 指令处。此时由于是 16 位代码，因此用 `i r` 命令（即 `info registers`）查看到寄存器都是 16 位系统上的寄存器。

然后我们用 `l start32` 查看 32 位代码的起始处，并用 `b 56` 将断点设置在 32 位代码的第一条汇编语句处，再用 `i r` 命令查看。

## 2. bootasm.S

BIOS 装入 `bootblock` 后（位于 `0x7c00`）将会把控制权转移给 `xv6` 的 `bootblock`，从此 `xv6` 的启动代码开始接管系统启动过程。`bootblock` 是由  `bootasm.S` 和 `bootmain.c` 生产的。

`bootblock` 的启动过程分为实模式部分和保护模式部分。

### 2.1 实模式代码

`bootasm.S` 第 10 行指出这是 16 bit 代码，可运行于 `x86` 的实模式。。GNU AS 汇编器使用的汇编语言采用的是 AT&T 语法（Linux 环境下的通用标准），该语法和 Intel 语法不同。 

BIOS 在执行的时候会打开中断，但这时 BIOS 已经不执行了，所以它的中断向量表之类的也就不再起作用了。此时 `bootloader` 关闭中断（`bootasm.S` 第 13 行），后面将在合适的时机设置 `IDT` 表然后再打开中断。

第 15~19 行将 `DE`、`ES` 和 `SS` 三个段寄存器都清零了，第四个寄存器 `CS` 被 BIOS 设置为 0。

第 21~37 行的代码用于打开地址 `A20` 线，从而可以访问高于 1 MB 的物理内存，这是一个 PC 兼容性引起历史问题。在刚进入 `bootblock` 的时候，硬件处于实地址模式，在此模式下只能使用 16 位的寄存器，因此真正可寻址的范围是 20 位地址，也就是 1 M 的地址空间。实模式下硬件通过把段寄存器的值作为 segment 左移四位然后加上 offset 的方式来得到线性地址 （`segment >> 4` + `offset`）。现在要进入保护模式使用 32 地址了，因此需要启用物理地址高位。

 第 39~45 行为进入到保护模式做准备，因此需要设置 `GDT`，而且要保证代码 “无缝” 平滑地执行。此时通过 `lgdt` 指令将第 85 行 `gdtdesc` 地址处的数据装入 `GDTR` 寄存器。其中 `GDTR` 寄存器包括一个 16 bit（`.word`）表的长度，以及 32 bit（`.long`）的表起始地址。可以看出 `GDT` 表的起始地址在 `gdt`，而 `gdt` 地址处有三个 `GDT` 表项（第 80~83 行）。第一个表项使用 `SEG_NULLASM` 宏来生成，而后面两项用 `SEG_ASM` 宏来生 成，这两种宏定义于 `asm.h`。

1. 第一项用 `SEG_NULLASM` 宏生成的项是全 0
2. 第二项 `SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)` 对应 `bootblock` 代码段，起点为 `0x0`，长度为 `0xffffffff`，类型为 `STA_X|STA_R`（可执行可读）
3. 第三项 `SEG_ASM(STA_W, 0x0, 0xffffffff)` 对应 `bootblock` 数据段，起点为 `0x0`，长度为 `0xffffffff`，类型为 `STA_W`（可写）

设置好 `GDTR` 后，第 43~45 行通过 `CR0` 的 PE 位使处理器进入保护模式。保护模式程序地址是用逻辑地址来表示的，逻辑地址通常保存成 `segment:offset` 的形式。也就是说，此时所有段的起点都是 0，弱化了 `x86` 的段管理功能。Linux 页采用类似的方式来使用 `x86` 硬件的内存地址部件。

接下来通过第 51 行的长跳转指令 `ljmp $(SEG_KCODE<<3), $start32` 跳转到 `start32` 标号所在的代码处。其中目的地址的段为 `$(SEG_KCODE<<3)`（`SEG_KCODE` 定义在 `mmu.h`，取值为 1，需要左移 3 位的原因是段选择字的低三位不是索引号），也就是跳转目的地址的 `cs=2`（`GDT` 表的第 3 项，第一项编号为 0），即代码段。前面已知代码段起始地址为 0，`$start32` 标识的地址就是 `0+start32` 的取值，因此实际上 "无缝平滑" 地跳转到标号 `start32` 位置（即 54 行的代码）。

虽然前面的 16 bit 的代码，后面是 32 bit 代码（使用 `.code32` 指示编译器按 32 位编译），但运行起来就如前后两条指令连续运行一般（从第 51 行顺序运行到第 54 行）。 

至此，开始了 32 位保护模式的代码运行阶段。

### 2.2 保护模式代码

进入保护模式后，第 55~62 行代码是设置各个段的索引，可以看到 `ds`、`es`、`ss` 三个选择字都索引到了数据段（编号为 2 的 `SEG_KDATA` 段），`fs` 和 `gs` （GS 段后面会被用作 “每 CPU 变量”（`cpu` 和 `proc`）的特殊段）选择字则指向无效段（编号为 0 的 `SEGNULLASM`）。启动保护模式后的代码段和数据段都映射到 `0~0xffffffff` 的线性地址范围。通过这样的设置使得 `xv6` 可以无视分段机制的地址映射问题，将逻辑地址和线性地址等效地使用。也就是说无论是用 `cs` 的取指令、`ds/es` 的访问数据或用 `ss` 的访问堆栈，段的起点都是 0，起关键作用的是各自相应的段内偏移地址。 

- 段寄存器设置为 0，32 位地址空间的布局如下

| 地址          | 描述               |
| ------------- | ------------------ |
| `0x0000`      | 起始地址           |
| `0x7c00`      | 启动扇区的起始地址 |
| `0x7d00`      | 启动扇区的结束地址 |
| `0xffff-ffff` | 结束地址           |

然后 `bootasm.S` 的第 65 行设置堆栈指针 `esp` 为 `$start`（即 `bootblock` 的第一行代码位置，定义在 `bootasm.S` 的第 11 行），由于 BIOS 将 `bootblock` 装载到 `0x7c00` 地址处，也就是说 `start` 地址就是 `0x7c00`，由于 `bootblock` 只站 1 个扇区共 512 字节，因此占用地址空间为 `0x7c00~0x7d00`。然后跳到 C 代码执行 `bootmain()`，`bootblock` 的汇编部分至此结束。

正常执行 `bootmain()` 是不可能返回的，因此如果执行到了 `bootasm.S` 第 70 行的代码，则说明系统出错了，例如后面 `bootmain()` 读入磁盘数据后发现不是 ELF 格式（即没有发现有效的内核）。这些代码执行两个 `outw` 指令的 IO 操作，向 `0x8a00` 端口写入数据从而引发 `Bochs` 虚 拟机的 `breakpoint`（真实机器在 `0x8a00` 地址只是普通内存没有任何特定作用），然后进入无限循环。 

至此，结束了  `bootasm.S` 中汇编代码的分析，开始第二步骤 C 语言的 `bootmain()` 代码运行阶段。

### 2.3 调试

由于 `xv6` 的默认设置中，其 `gdb` 初始化脚本 `.gdbinit` 的最后一行的命令指出所使用的符号表是 `kernel`，因此是无法处理 `bootblock` 的代码和符号的。 

因此我们由两种方法解决：一是将最后的一行从 `symbol-file kernel` 修改为 `symbol-file bootblock.o` 或 `file bootblock.o`；二是直接在 `gdb` 启动后执行 `file bootblock.o` 命令，替换调试目标程序以及符号表。 

此时再执行 `gdb` 调试，可以查看 `bootblock` 的信息。我们执行 `l start` 查看 `bootblock` 最初的几条指令，并用 `p/x $cs` 查看到 `cs=0xf000`，用 `p/x $eip` 查看到 `eip=0xfff0`，正处于 PC 还未执行 BIOS 的时候。 

我们用 `b start` 将断点设置在 `bootblock` 的第一条指令，并用 `c` 命令执行到该断点，检查看到 `cs=0x0`、`eip=0x7c00`。此时正是 BIOS 刚跳转到 `bootblock` 的第一条指令而将控制权转交给 `bootblock` 的时刻。 

到保护模式后，`gdb` 因权限不够而无法读写 `gdt` 等寄存器。此时需要用 `qemu` 的调试功能，我们需要将 `xv6` 默认的 `Makefile` 做一点小修改，将其中 `qemu-nox-gdb` 目标的规则修改成：

```makefile
emu-gdb: fs.img xv6.img .gdbinit
        @echo "*** Now run 'gdb'." 1>&2
#       $(QEMU) -serial mon:stdio $(QEMUOPTS) -S $(QEMUGDB)
        $(QEMU) -monitor stdio $(QEMUOPTS) -S $(QEMUGDB)
```

然后运行 `make qemu-gdb`，此时在 shell 窗口中的内容和之前不同，出现了 `qemu` 命令提示。

接着就在此窗口下输入命令 `info registers`，可以将整个系统的全部寄存器信息打印出来。（后面有练习要求读者调试跟踪到保护模式代码处，并检查相应的程序段描述符信息）。

### 2.4 代码回顾

此时我们已经对 `bootasm.S` 的代码已经有了完整的了解。其中的第11行定义了一个全局变量 `start`，即 `xv6` 的第一条指令 `cli` 的地址。第 54 行的 `start32` 开始进入保护模式代码，并通过设置段寄存器构造出 `0~0xffff-ffff` 的线性地址范围。最后通过 66 行的 `call bootmain` 跳入到 `bootmain.c` 中的 `bootmain()` 函数。正常情况是不会从 `bootmain()` 返回的，在 `call bootmain` 之后的代码对应于异常情况。 

