---
title: 2. 启动扇区和内核
---

XV6 二进制代码分为两部分：

1. 启动扇区的代码 `bootblock` 
2. 内核代码 `kernel` 

由于都是 Linux 系统下使用 GCC 工具生成的代码，因此都是使用 ELF 格式的目标文件，可以用 `binutils` 工具查看和分析。

## 1. 启动扇区

在 X86 PC 启动的时候，首先执行的代码是主板上的 BIOS （Basic Input Output System）。BIOS 存放在非易失存储器中，主要完成一些硬件自检的工作。在这些工作做完之后，BIOS 会从启动盘里读取第一个扇区（启动扇区 boot sector）的 512 字节数据到内存中，这 512 字节的代码就是我们熟知的 `bootloader` 。在导入完成后 CPU 控制权由 BIOS 转交给 `bootloader` 。BIOS 会把 `bootloader` 导入到 `0x7c00` 开始的地方，然后把 PC 指针设成此地址（通过设置寄存器 `%ip`），将控制权交给 `bootloader` 。

XV6 启动扇区的代码 `bootblock`  就是扮演上述 `bootloader` 的角色，它将继续负责将 XV6 的内核代码 `kernel` 装载到内存，并将控制权交给 `kernel` ，从而完成启动过程。

### 1.1 启动扇区的生成

XV6 引导加载器 `bootblock` 包括两个源文件，一个由 16 位和 32 位汇编混合编写而成的  [bootasm.S](https://github.com/professordeng/xv6-expansion/blob/master/bootasm.S)，另一个由 C 写成 [bootmain.c](https://github.com/professordeng/xv6-expansion/blob/master/bootmain.c) 。

XV6 系统通过 `bootasm.S`和 `bootmain.c` 生成 `bootblock.o` 目标文件后，再通过 `objcopy` 工具将其中的代码节 `.text` 抽取出来写到 `bootblock` 文件中产生启动扇区。

启动扇区 `bootblock` 的生成步骤包括编译、链接与定制。如下代码是 `Makefile` 中生成 `bootblock` 的部分：

```bash
bootblock: bootasm.S bootmain.c
    $(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
    $(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
    $(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
    $(OBJDUMP) -S bootblock.o > bootblock.asm
    $(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
    ./sign.pl bootblock
```

其中变量 `CC` 是编译器，`LD` 是链接器、`OBJDUMP` 是 `objdump` 工具、`OBJCOPY` 是 `objcopy` 工具。

编译中使用了 `-nostdinc` 参数，编译器将不在系统头文件路径下搜索。这是因为 XV6 并没有实现标准 C 语言库，由 `-I.` 指出当前路径为头文件搜索路径，可以找到例如 XV6 内核头文件 [defs.h](https://github.com/professordeng/xv6-expansion/blob/master/defs.h) 等。

### 1.2 bootblock.o

执行 `readelf -l bootblock.o` 查看段的情况，内容如下

```bash
Elf file type is EXEC (Executable file)
Entry point 0x7c00
There are 2 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000074 0x00007c00 0x00007c00 0x0027c 0x0027c RWE 0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .eh_frame 
   01    
```

可以看到启动扇区 `bootblock.o` 只有编号为 00 段的类型为 `LOAD`，表示该段需要装入内存，长度为 `0x0027c` （大于一个扇区的长度），装入地址为 `0x7c00` ，该地址是 PC 的 BIOS 决定的。PC 启动后最先执行的是主板 BIOS 代码，它将启动扇区内容读入到内存的 `0x7c00` 位置，然后再跳转到启动代码的第一条指令。

在 `Section to Segment mapping` 下，看到 00 段是由 `.text` 节和 `.eh_frame` 节构成的。其中 `.text` 会被拷贝到 `bootblock` 文件中，而 `.eh_frame` 节将会被丢弃。

既然只有 `.text` 有用，我们继续用 `readelf -S bootblock.o` 查看 `bootblock.o` 的节的信息。可以看到 `.text` 是 `bootblock.o` 的代码，位于 `bootblock.o` 文件的 `0x74` 字节偏移的位置，大小为 `0x1c0` ，`Adrr` 列指出装入地址为 `0x7c00` （与 BIOS 约定相一致）。从 `Flg` 标志可以看出需要在内存分配空间（A）、访问权限为可执行（X），可写（W）。我们不关心其中有关异常处理的 `.eh_frame` 节，因为启动时有异常也没什么可以处理的。

```bash
There are 13 section headers, starting at offset 0x1220:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00007c00 000074 0001c0 00 WAX  0   0  4
  [ 2] .eh_frame         PROGBITS        00007dc0 000234 0000bc 00   A  0   0  4
  [ 3] .comment          PROGBITS        00000000 0002f0 00002b 01  MS  0   0  1
  [ 4] .debug_aranges    PROGBITS        00000000 000320 000040 00      0   0  8
  [ 5] .debug_info       PROGBITS        00000000 000360 00050b 00      0   0  1
  [ 6] .debug_abbrev     PROGBITS        00000000 00086b 0001e3 00      0   0  1
  [ 7] .debug_line       PROGBITS        00000000 000a4e 00012c 00      0   0  1
  [ 8] .debug_str        PROGBITS        00000000 000b7a 0001dd 01  MS  0   0  1
  [ 9] .debug_loc        PROGBITS        00000000 000d57 00022a 00      0   0  1
  [10] .symtab           SYMTAB          00000000 000f84 0001a0 10     11  18  4
  [11] .strtab           STRTAB          00000000 001124 00007c 00      0   0  1
  [12] .shstrtab         STRTAB          00000000 0011a0 00007f 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

### 1.3 bootblock

从 `Makefile` 中的 `bootblock` 的生成规则可知，它先通过 `objcopy` 将 `bootblock.o` 其中的 `.text` 节拷贝到 `bootblock` 中（此时长度为 448 字节，小于一个扇区）。然后再通过 [sign.pl](https://github.com/professordeng/xv6-expansion/blob/master/sign.pl) 的一个 `perl` 脚本规整为一个磁盘扇区的 512 字节大小，并将最后两个字节填写上 `0x55` 和 `0xaa`。

## 2. 内核代码

`bootblock` 的主要任务就是将 XV6 内核 `kernel` 文件读入，并将控制权转交给 `kernel` 代码，从而运行 XV6 操作系统。`kernel` 代码本身又包括主体代码和辅助初始化代码两部分。

1. 辅助初始化代码

   包括 [initcode.S](https://github.com/professordeng/xv6-expansion/blob/master/initcode.S)（生成 `initcode`）和 [entryother.S](https://github.com/professordeng/xv6-expansion/blob/master/entryother.S)（生成 `entryother`）。`initcode` 是第一个进程的前身，`entryother` 是其他 CPU 核启动时的初始化代码。

2. 主体代码

   [entry.S](https://github.com/professordeng/xv6-expansion/blob/master/entry.S) 负责从 X86 的线性地址模式转向分页的虚地址模式、建立页表映射等内核启动的初始化工作。

   由 `$OBJS` 表示的 [proc.c](https://github.com/professordeng/xv6-expansion/blob/master/proc.c)、[fs.c](https://github.com/professordeng/xv6-expansion/blob/master/fs.c)、[vm.c](https://github.com/professordeng/xv6-expansion/blob/master/vm.c) … [ide.c](https://github.com/professordeng/xv6-expansion/blob/master/ide.c) 等文件，由脚本 [kernel.ld](https://github.com/professordeng/xv6-expansion/blob/master/kernel.ld) 生成 `kernel` 的主体部分。`$(OBJS)` 所对应的 XV6 内核主体就是运行在分页地址模式之下，使用页表映射的虚拟内存。

内核代码生成的文件布局大致如下:

|                    | kernel 主体 | initcode   | entryother |
| ------------------ | ----------- | ---------- | ---------- |
| 文件偏移（offset） | 0x00        | 0xa460     | axa516     |
| 物理地址（PA）     | 0x00100000  | 0x10a460   | 0x10a48c   |
| 虚拟地址（VA）     | 0x80100000  | 0x8010a460 | 0x8010a48c |

### 2.1 kernel 的生成

XV6 内核主体代码由许多独立的 C 源文件编译、链接而成，其中主要的源代码用于实现操作系统的进程管理、内存管理、文件系统管理和设备管理等，再加上辅助的初始代码等。

**编译**

查看 `Makefile` 中 `kernel` 目标，如下：

```makefile
kernel: $(OBJS) entry.o entryother initcode kernel.ld
    $(LD) $(LDFLAGS) -T kernel.ld -o kernel entry.o $(OBJS) -b binary initcode entryother
    $(OBJDUMP) -S kernel > kernel.asm
    $(OBJDUMP) -t kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > kernel.sym
```

发现其依赖于 `$(OBJS)`、`entry.o`、`entryother`、`initcode` 和 `kernel.ld`。其中 `$(OBJS)` 变量包含了内核所需的主要目标文件名，另外的 `entry.o`、`entryother` 和 `initcode` 属于内核的启动代码，最后的 `kernel.ld` 是用于描述链接布局和内存定位的链接脚本。

后面的学习中，主要精力就是对 `$(OBJS)` 所对应的 C 代码进行分析，操作系统的四大管理功能都由它们实现。

内核符号表通过 `objdump` 工具输出到 `kernel.sym` 文件中，用于记录所有内核符号和对应的地址。在将来调试中分析某个地址的时候，可以从中获得参考价值。

利用 `wc -l kernel.sym` 命令统计得到 `kernel.sym` 中有 518 行（每行一个符号）。用 `head kernel.sym` 命令查看，其内容分为两列，第一列是地址，第二列是对应的符号，前十行内容如下所示。

```bash
80100000 .text
80106ec0 .rodata
801078ac .stab
801078ad .stabstr
80108000 .data
8010a520 .bss
00000000 .debug_line
00000000 .debug_info
00000000 .debug_abbrev
00000000 .debug_aranges
```

**链接**

链接过程中，`entry.o` 和 `$(OBJS)` 按照 `kernel.ld` 链接脚本的要求进行链接，而且将参数 `-b binary` 之后的 `initcode` 和 `entryother` 按照二进制方式安排布局在指定地址。也就是说 `initcode` 和 `entryother` 是已经经过链接布局的，并不需要像 `$(OBJS)` 和 `entry.o` 那样（会将同类的 `.text`、 `.data`、`.bss` 进行合并布局），而是按照自己原有的 ELF 格式进行独立布局，并拼接在 `entry.o` 和 `$(OBJS)` 链接结果的后面。

观察链接器脚本 [kernel.ld](https://github.com/professordeng/xv6-expansion/blob/master/kernel.ld)，发现其内核代码的地址布局从 [0x80100000](https://github.com/professordeng/xv6-expansion/blob/master/kernel.ld#L14) 开始的。但是其代码的装载地址是 [0x100000](https://github.com/professordeng/xv6-expansion/blob/master/kernel.ld#L12) 。后面解释装载地址和运行地址差异。

其中 [ENTRY(_start)](https://github.com/professordeng/xv6-expansion/blob/master/kernel.ld#L6) 命令将 [_start](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L41) 作为入口地址（`_start` 在 `entry.S` 中定义，取值为 `0x0010000c`。该地址将记录在 ELF 文件头结构体 `elfhdr` 的 `entry` 成员中（即 `elfhdr.entry`）。

用 `readelf -l kernel` 查看链接输出的 `kernel`，可以看到只有一个编号为 00 的段需要载入， `FileSiz` 刚好是 `0x0a516`（正好是 `entryother` 开始的地址）。

```bash
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

### 2.2 辅助初始化代码

在 `Makefile` 中相关的代码如下

```makefile
entryother: entryother.S
    $(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c entryother.S
    $(LD) $(LDFLAGS) -N -e start -Ttext 0x7000 -o bootblockother.o entryother.o
    $(OBJCOPY) -S -O binary -j .text bootblockother.o entryother
    $(OBJDUMP) -S bootblockother.o > entryother.asm

initcode: initcode.S
    $(CC) $(CFLAGS) -nostdinc -I. -c initcode.S
    $(LD) $(LDFLAGS) -N -e start -Ttext 0 -o initcode.out initcode.o
    $(OBJCOPY) -S -O binary initcode.out initcode
    $(OBJDUMP) -S initcode.o > initcode.asm
```

`kernel` 文件中包含了一部分代码用于 `init` 进程的初始化、其他处理器的启动初始化的代码。其中 `init`  进程的代码将被装载到 `0x0000` 地址，而其他核启动初始代码将装入 `0x7000` 地址处。

其中的 `entryother` 刚开始随着内核主体代码一起装入到 `0x100000` 地址。在主处理器启动完成之后，`entryother` 将被拷贝到 `0x7000` 地址（和链接时的地址一致）给其他处理器作为启动代码。

其中的 `initcode` 在启动时随内核主体代码一起装入到 `0x100000`。创建第一个进程时，拷贝到用户空间 0 地址处作为第一个进程（`init` 进程）的内存影像。XV6 的进程代码是按照 0 地址（虚地址）开始布局的。