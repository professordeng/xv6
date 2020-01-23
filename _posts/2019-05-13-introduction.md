---
title: xv6 概述
date: 2019-04-09
---

`xv6` 是 `Unix` 操作系统的一种简化实现，因此 `xv6` 的基本概念和 `Unix` 同源。下面内容要求读者对 `Unix` 或 `Linux` 有基本的了解。因为我们着重于对这些概念的代码实现。如果不熟悉 `Unix`，建议先学习一下操作系统的知识和 `Linux/Unix` 的核心概念。学习过程中如有拓展有关 OS 开发的软硬件知识的疑问，可以在 [OSDev](https://wiki.osdev.org/) 网站上查找资料并参与讨论。

下面先简单了解一下 `xv6` 所用到的启动扇区 `bootblock` 文件、内核代码 `kernel` 文件和磁盘镜像文件。然后学习 `xv6` 进程管理与调度、内存管理、文件系统和设备的基本概念。

## 1. 代码总览

`xv6` 的源代码总量较小，利用 `ls` 命令可以查看 `xv6` 的头文件、C 语言源文件和汇编代码文件

```bash
➜  xv6-expansion git:(dev) ls *.S
bootasm.S  entryother.S  entry.S  initcode.S  swtch.S  trapasm.S  usys.S
➜  xv6-expansion git:(dev) ls *.c
bio.c       file.c      ioapic.c  log.c     mp.c      sh.c         sysfile.c  usertests.c
bootmain.c  forktest.c  kalloc.c  ls.c      picirq.c  sleeplock.c  sysproc.c  vm.c
cat.c       fs.c        kbd.c     main.c    pipe.c    spinlock.c   trap.c     wc.c
console.c   grep.c      kill.c    memide.c  printf.c  stressfs.c   uart.c     zombie.c
echo.c      ide.c       lapic.c   mkdir.c   proc.c    string.c     ulib.c
exec.c      init.c      ln.c      mkfs.c    rm.c      syscall.c    umalloc.c
➜  xv6-expansion git:(dev) ls *.h
asm.h   defs.h   file.h  memlayout.h  param.h      spinlock.h  traps.h  x86.h
buf.h   elf.h    fs.h    mmu.h        proc.h       stat.h      types.h
date.h  fcntl.h  kbd.h   mp.h         sleeplock.h  syscall.h   user.h
```

另外还有一些辅助性的代码以及 `Makefile` 等编译有关的文件。

## 2. 二进制代码与镜像

`xv6` 二进制代码分为两部分：

1. 是启动扇区的代码 `bootblock` 
2. 内核代码 `kernel` 

由于都是 Linux 系统下使用 `gcc` 工具生成的代码，因此都是使用 `ELF` 格式的目标文件，可以用 `binutils` 工具查看和分析。

### 2.1 启动扇区

在  `x86 PC` 启动的时候，首先执行的代码是主板上的 BIOS （Basic Input Output System），主要完成一些硬件自检的工作。在这些工作做完之后，BIOS 会从启动盘里读取第一个扇区（启动扇区 boot sector）的 512 字节数据到内存中，这 512 字节的代码就是我们熟知的 `bootloader` 。在导入完成后 CPU 控制权由 BIOS 转交给 `bootloader` 。BIOS 会把 `bootloader` 导入到 `0x7c00` 开始的地方，然后把 PC 指针设成此地址，将控制权交给 `bootloader` 。

`xv6` 启动扇区的代码 `bootblock`  就是扮演上述 `bootloader` 的角色，它将继续负责将 `xv6` 的内核代码 `kernel` 装载到内存，并将控制权交给 `kernel` ，从而完成启动过程。

#### 启动扇区的生成

`xv6` 系统通过 `bootasm.S` 和 `bootmain.c` 生成 `bootblock.o` 目标文件后，再通过 `objcopy` 工具将其中的代码节 `.text` 抽取出来到 `bootblock` 文件中产生启动扇区。

启动扇区 `bootblock` 的生成步骤包括编译、链接与定制。如下代码是 `Makefile` 中生成 `bootblock` 的部分：

```makefile
bootblock: bootasm.S bootmain.c
    $(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
    $(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
    $(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
    $(OBJDUMP) -S bootblock.o > bootblock.asm
    $(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
    ./sign.pl bootblock
```

其中变量 `CC` 是编译器，`LD` 是链接器、`OBJDUMP` 是 `objdump` 工具、`OBJCOPY` 是 `objcopy` 工具。

编译中使用了 `-nostdinc` 参数，编译器将不在系统头文件路径下搜索。这是因为 `xv6` 并没有实现标准 C 语言库，由 `-I.` 指出当前路径为头文件搜索路径，可以找到例如 `xv6` 内核头文件 `defs.h` 等。

#### bootblock.o

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

可以看到启动扇区 `bootblock.o` 只有编号为 00 段的类型为 `LOAD`，表示该段需要装入内存，长度为 `0x0027c` （大于一个扇区的长度），装入地址为 `0x7c00` ，该地址是 PC 的 BIOS 决定的。 PC 启动后最先执行的是主板 BIOS 代码，它将启动扇区内容读入到内存的 `0x7c00` 位置，然后再跳转到启动代码的第一条指令。

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

#### bootblock

从 `Makefile` 中的 `bootblock` 的生成规则可知，它先通过 `objcopy` 将 `bootblock.o` 其中的 `.text` 节拷贝到 `bootblock` 中（此时长度为 448 字节，小于一个扇区）。然后再通过 `sign.pl` 的一个 `perl` 脚本规整为一个磁盘扇区的 512 字节大小，并将最后两个字节填写上 `0x55` 和 `0xaa`。`sign.pl` 代码如下：

```perl
#!/usr/bin/perl 
  
open(SIG, $ARGV[0]) || die "open $ARGV[0]: $!";       # 打开命令行参数指出的文件

$n = sysread(SIG, $buf, 1000);                  # 读入 1000 字节到 buf 中

if($n > 510){                                  # 如果读入字节数大于 510
  print STDERR "boot block too large: $n bytes (max 510)\n"; # 提示超过一个扇区
  exit 1;                      
}

print STDERR "boot block is $n bytes (max 510)\n"; # 打印提示: 启动扇区有效字节数

$buf .= "\0" x (510-$n);           
$buf .= "\x55\xAA";     # 最后两个字节填入 0x55、0xaa

open(SIG, ">$ARGV[0]") || die "open >$ARGV[0]: $!";
print SIG $buf;                 # 将缓冲区内容写回文件
close SIG;
```

### 2.2 内核代码

`bootblock` 的主要任务就是将 `xv6` 内核 `kernel` 文件读入，并将控制权转交给 `kernel` 代码，从而运行 `xv6` 操作系统。`kernel` 代码本身又包括主体代码和辅助初始化代码两部分。

1. 辅助初始化代码

   包括 `initcode.S` 和 `entryother.S`，生成 `initcode` 和 `entryother`。`initcode` 部分是第一个进程的前身，`entryother` 是其他 CPU 核启动时的初始化代码。

2. 主体代码

   - `entry.S` ，负责从 `x86` 的线性地址模式转向分页的虚地址模式、建立页表映射等内核启动的初始化工作。
   - 由 `$OBJS` 表示的 `proc.c`、`fs.c`、`vm.c` … `ide.c` 等文件，由脚本 `kernel.ld` 生成 `kernel` 的主体部分。`$(OBJS)` 所对应的 `xv6` 内核主体就是运行在分页地址模式之下，使用页表映射的虚拟内存。

内核代码生成的文件布局大致如下

|                    | kernel 主体 | initcode   | entryother |
| ------------------ | ----------- | ---------- | ---------- |
| 文件偏移（offset） | 0x00        | 0xa460     | axa516     |
| 物理地址（PA）     | 0x00100000  | 0x10a460   | 0x10a48c   |
| 虚拟地址（VA）     | 0x80100000  | 0x8010a460 | 0x8010a48c |

#### kernel 的生成

`xv6` 内核主体代码由许多独立的 C 源程序编译、链接而成，其中主要的源代码用于实现操作系统的进程管理、内存管理、文件系统和设备等，再加上辅助的初始代码等。

- 编译

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

  利用 `wc -l kernel.sym` 命令统计得到 `kernel.sym` 中个数为 518 个。用 `head kernel.sym` 命令查看，其内容分为两列，第一列是地址，第二列是对应的符号，前十行内容如下所示。

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

- 链接

  链接过程中，`entry.o` 和 `$(OBJS)` 按照 `kernel.ld` 链接脚本的要求进行链接，而且将参数 `-b binary` 之后的 `initcode` 和 `entryother` 按照二进制方式安排布局在指定地址。也就是说 `initcode` 和 `entryother` 是已经经过链接布局的，并不需要像 `$(OBJS)` 和 `entry.o` 那样（会将同类的 `.text .data .bss` 进行合并布局），而是安装自己原有的 `ELF` 格式进行独立布局，并拼接在 `entry.o` 和 `$(OBJS)` 链接结果的后面。
  
  注意观察链接器脚本 `kernel.ld` ，内容如下
  
  ```
  /* Simple linker script for the JOS kernel.
     See the GNU ld 'info' manual ("info ld") to learn the syntax. */
  
  OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
  OUTPUT_ARCH(i386)
  ENTRY(_start)
  
  SECTIONS
  {
  	/* Link the kernel at this address: "." means the current address */
          /* Must be equal to KERNLINK */
  	. = 0x80100000;
  
  	.text : AT(0x100000) {
  		*(.text .stub .text.* .gnu.linkonce.t.*)
  	}
  
  	PROVIDE(etext = .);	/* Define the 'etext' symbol to this value */
  
  	.rodata : {
  		*(.rodata .rodata.* .gnu.linkonce.r.*)
  	}
  
  	/* Include debugging information in kernel memory */
  	.stab : {
  		PROVIDE(__STAB_BEGIN__ = .);
  		*(.stab);
  		PROVIDE(__STAB_END__ = .);
  		BYTE(0)		/* Force the linker to allocate space
  				   for this section */
  	}
  
  	.stabstr : {
  		PROVIDE(__STABSTR_BEGIN__ = .);
  		*(.stabstr);
  		PROVIDE(__STABSTR_END__ = .);
  		BYTE(0)		/* Force the linker to allocate space
  				   for this section */
  	}
  
  	/* Adjust the address for the data segment to the next page */
  	. = ALIGN(0x1000);
  
  	/* Conventionally, Unix linkers provide pseudo-symbols
  	 * etext, edata, and end, at the end of the text, data, and bss.
  	 * For the kernel mapping, we need the address at the beginning
  	 * of the data section, but that's not one of the conventional
  	 * symbols, because the convention started before there was a
  	 * read-only rodata section between text and data. */
  	PROVIDE(data = .);
  
  	/* The data segment */
  	.data : {
  		*(.data)
  	}
  
  	PROVIDE(edata = .);
  
  	.bss : {
  		*(.bss)
  	}
  
  	PROVIDE(end = .);
  
  	/DISCARD/ : {
  		*(.eh_frame .note.GNU-stack)
  	}
  }
  
  ```
  
  发现其内核代码的地址布局从 `0x80100000` 开始的。但是其代码的装载地址是 `0x100000` 。后面解释装载地址和运行地址差异。
  
  其中 `ENTRY(_start)` 命令将 `_start` 作为入口地址（`_start` 符号在 `entry.S` 中定义，取值为 `0x0010000c`。该地址将记录在 ELF 文件头结构体 `elfhdr` 的 `entry` 成员中（即 `elfhdr.entry`）。
  
  `ENTRY(_start)` 中 `_start` 表示入口地址，在 `entry.S` 中定义取值为 `0x0010000c` 记录在 ELF 文件的文件头结构体 `elfhdr` 的 `entry` 成员中（即 `elfhdr.entry`）。
  
  用 `readelf -l kernel` 查看链接输出的 `kernel`，可以看到只有一个编号为 00 的段需要载入， `FielSiz` 刚好是 `0x0a516`（正好是 `entryother` 结束的地址）。
  
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

#### 辅助初始化代码

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

其中的 `initcode` 在启动时随内核主体代码一起装入到 `0x100000`。创建第一个进程时，拷贝到用户空间 0 地址处作为第一个进程（`init` 进程）的内存镜像。`xv6` 的进程代码是按照 0 地址（虚地址）开始布局的。

#### kernelmemfs 的生成

虽然大多数情况下 `xv6` 使用 `IDE` 磁盘作为文件系统载体，但是某些情况下也会使用内存盘作为文件系统的载体。当使用内存盘的时候，对应的内核不是 `kernel` 文件，而是 `kernelmemfs` 。该内核将完整的磁盘文件系统镜像包含进来。当启动扇区装载这个 `kernelmemfs` 内核的时候，已经把整个文件系统一起装进内存，因此后续的磁盘操作都按内存操作方式来完成。`kernelmemfs` 的生成过程如下所示，与 `kernel` 的生成过程类似。

1. kernel 主体部分

   `pro.o`、`fs.o`、`vm.o` 和 `memide.o` 组成的变量 `$(MEMFSOBJS)` 、`entry.o` 、`initcode` 和 `entryother`。

2. `fs.img` 组成内存中的磁盘文件系统。

下面讲解形成 `kernelmemfs` 的具体细节 

1. 编译

   当用 `kernelmemfs` 镜像时，内核的主体代码不是由 `$(OBJS)` 构成的，而是用 `$(MEMFSOBJS)`（将 `$(OBJS)` 去除1 `ide.o` 并替换为 `memide.o`）构成的。因为此时磁盘文件系统是使用内存模拟的，因此文件系统读写磁盘时不再使用 `ide.o` 中的代码，而是利用 `memide.o` 中的代码。

   ```makefile
   # kernelmemfs is a copy of kernel that maintains the
   # disk image in memory instead of writing to a disk.
   # This is not so useful for testing persistent storage or
   # exploring disk buffering implementations, but it is
   # great for testing the kernel on real hardware without
   # needing a scratch disk.
   MEMFSOBJS = $(filter-out ide.o,$(OBJS)) memide.o
   kernelmemfs: $(MEMFSOBJS) entry.o entryother initcode kernel.ld fs.img
       $(LD) $(LDFLAGS) -T kernel.ld -o kernelmemfs entry.o  $(MEMFSOBJS) -b binary initcode entryother fs.img
       $(OBJDUMP) -S kernelmemfs > kernelmemfs.asm
       $(OBJDUMP) -t kernelmemfs | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > kernelmemfs.sym
   ```

2. 链接

   `kernelmemfs` 的链接和 `kernel` 类似，也是使用 `kernel.ld` 链接脚本，因此具有相同的内存和布局。不同之处在于 `-b binary` 后面多了个 `fs.mg` （即磁盘镜像和内核链接成一个单一的 `ELF` 文件。其中 `fs.img` 是磁盘文件系统的镜像（`fs.img` 的生成过程会在后面分析）。

   执行 `make kernelmemfs` ，然后比较 `kernelmemfs` 和 `kernel` 的大小，发现 `kernelmemfs` 远大于 `kernel` 文件，和我们预想一样。

   ```bash
➜  xv6-expansion git:(dev) ls -l kernelmemfs
   -rwxrwxr-x 1 ubuntu ubuntu 688100 Jan 23 13:14 kernelmemfs
➜  xv6-expansion git:(dev) ls -l kernel     
   -rwxrwxr-x 1 ubuntu ubuntu 178904 Jan 19 17:26 kernel
   ```
   
   接着我们通过 `readelf -l kernelmemfs` 查看 `kernelmemfs` 的段和节的情况，只能看出需要 `LOAD` 装入的 00 段比较大（对比 `kernel` 的 `0x0a516`）
   
   ```bash
   Elf file type is EXEC (Executable file)
   Entry point 0x10000c
   There are 3 program headers, starting at offset 52
   
   Program Headers:
     Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
     LOAD           0x001000 0x80100000 0x00100000 0x076af 0x076af R E 0x1000
     LOAD           0x0086af 0x801076af 0x801076af 0x7fe67 0x8adb9 RW  0x1000
     GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10
   
    Section to Segment mapping:
     Segment Sections...
      00     .text .rodata 
      01     .stab .stabstr .data .bss 
      02     
   ```
   
   但是继续利用 `readelf -S kernelmemfs` 查看 `kernelmemfs` 的节，从 `.data` 节的大小我们可以知道 `fs.img` 是以数据的形式并入到 `.data` 节（`Size` 为 `07f516`）。
   
   ```bash
   There are 18 section headers, starting at offset 0xa7d14:
   
   Section Headers:
     [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
     [ 0]                   NULL            00000000 000000 000000 00      0   0  0
     [ 1] .text             PROGBITS        80100000 001000 006cb3 00  AX  0   0 16
     [ 2] .rodata           PROGBITS        80106cc0 007cc0 0009ef 00   A  0   0 32
     [ 3] .stab             PROGBITS        801076af 0086af 000001 0c  WA  4   0  1
     [ 4] .stabstr          STRTAB          801076b0 0086b0 000001 00  WA  0   0  1
     [ 5] .data             PROGBITS        80108000 009000 07f516 00  WA  0   0 4096
     [ 6] .bss              NOBITS          80187520 088516 00af48 00  WA  0   0 32
     [ 7] .debug_line       PROGBITS        00000000 088516 002503 00      0   0  1
     [ 8] .debug_info       PROGBITS        00000000 08aa19 010066 00      0   0  1
     [ 9] .debug_abbrev     PROGBITS        00000000 09aa7f 003853 00      0   0  1
     [10] .debug_aranges    PROGBITS        00000000 09e2d8 0003a8 00      0   0  8
     [11] .debug_str        PROGBITS        00000000 09e680 000e3a 01  MS  0   0  1
     [12] .debug_loc        PROGBITS        00000000 09f4ba 004e94 00      0   0  1
     [13] .debug_ranges     PROGBITS        00000000 0a434e 0006a0 00      0   0  1
     [14] .comment          PROGBITS        00000000 0a49ee 00002b 01  MS  0   0  1
     [15] .symtab           SYMTAB          00000000 0a4a1c 002080 10     16  78  4
     [16] .strtab           STRTAB          00000000 0a6a9c 0011d1 00      0   0  1
     [17] .shstrtab         STRTAB          00000000 0a7c6d 0000a5 00      0   0  1
   Key to Flags:
     W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
     L (link order), O (extra OS processing required), G (group), T (TLS),
     C (compressed), x (unknown), o (OS specific), E (exclude),
     p (processor specific)
   ```
   
   再进一步，用 `readelf -s kernelmemfs | grep fs.img` 查看 `kernelmemfs` 的符号表，发现在数据节（5 号节）有 `_binary_fs_img_start` ，这是连接器以 `binary` 方式将 `fs.img` 链接进来的时候创建的符号。其地址位于 `0x8010a516 `，正好是在原来 `kernel` 结束的地方。（kernel 大小为 `0x516`），另外 `fs.img` 的结束位置符号 `_binary_fs_end=80187516` 。链接器也为 `fs.img` 的大小创建了符号 ` ABS _binary_fs_img_size=0007d000` ，正好等于 `fs.img` 的大小 `0x7d000 = 51200` 。
   
   ```bash
   121: 8010a516     0 NOTYPE  GLOBAL DEFAULT    5 _binary_fs_img_start
   255: 0007d000     0 NOTYPE  GLOBAL DEFAULT  ABS _binary_fs_img_size
   460: 80187516     0 NOTYPE  GLOBAL DEFAULT    5 _binary_fs_img_end
   ```

### 2.3 磁盘镜像

在内核启动之后，需要有文件系统保持程序和数据，例如 `shell` 中所执行的外部指令：`ls`、`mkdir`、`rm`、`cp` 等程序，以及用户所需的其他数据文件等。由于我们将 `xv6` 运行于 `QEMU` 仿真环境，因此需要把相应的文件系统内容形成磁盘镜像文件。

`xv6` 中有两个磁盘镜像，一个是 `xv6.img` ，用于存放 `xv6` 操作系统；另一个是 `fs.img`，作为磁盘文件系统的镜像。当使用 `qemu` 仿真器运行 `xv6` 是，将使用上述两个镜像。`Makefile` 中定义了 `QEMUOPTS` 变量，用作 `qemu` 仿真参数，其中使用 `-driver` 选项定义了两个磁盘驱动器：

```makefile
QEMUOPTS = -drive file=fs.img,index=1,media=disk,format=raw -drive file=xv6.img,index=0,media=disk,format=raw -smp $(CPUS) -m 512 $(QEMUEXTRA)
```

1. 第一个驱动器编号为 0，镜像文件使用的是 `xv6.img` ，格式为 `raw-smp`
2. 第二个驱动器编号为 1，镜像文件使用的是 `fs.img`，格式为 `raw`

`QEMUOPS` 中的 `QEMUEXTRA` 暂时没有定义，根据名字可知：读者可以根据自己的需要扩充其他额外的选项。

#### QEMU 仿真中的磁盘镜像

例如 `make qemu` 时，不仅包含上面选项，还包括 `-serial mon:stdio` 参数。而 `make qemu-nox` 虽然还是使用上面的选项，但是多了一个 `-nographic` 选项，因此不会弹出额外的窗口，而是直接在现有终端下显示 `xv6` 的输出。`make qemu-memfs` 则是使用内存盘来保存整个文件系统。

```makefile
qemu: fs.img xv6.img
    $(QEMU) -serial mon:stdio $(QEMUOPTS)

qemu-memfs: xv6memfs.img
    $(QEMU) -drive file=xv6memfs.img,index=0,media=disk,format=raw -smp $(CPUS) -m 256

qemu-nox: fs.img xv6.img
    $(QEMU) -nographic $(QEMUOPTS)
```

综上所述，我们可以得到：

1. `xv6.img` 由 `bootblock` 和 `kernel` 构成。
2. `fs.img` 由 `README.md` 和 `$(UPROGS)` 构成。
3. `xv6memfs.img` 由 `bootblock` 和 `kernelmemfs` 构成。

#### xv6.img 镜像

从 `Makefile` 有关 `xv6.img` 的生成规则看，`xv6.img` 依赖于启动扇区 `bootblock` 以及 `kernel`。

```makefile
xv6.img: bootblock kernel
    dd if=/dev/zero of=xv6.img count=10000
    dd if=bootblock of=xv6.img conv=notrunc
    dd if=kernel of=xv6.img seek=1 conv=notrunc
```

#### fs.img 镜像

磁盘镜像 `fs.img` 是通过 `mkfs` 工具完成的，相应的 `Makefile` 给规则如下：

```makefile
fs.img: mkfs README.md $(UPROGS)
    ./mkfs fs.img README.md $(UPROGS)
```

也就是说 `mkfs` 将 `README.md` 和 `$(UPROGS)` 指定的那些文件创建出符合 `xv6` 文件系统格式的磁盘镜像。`$(UPROGS)` 给出的文件名对应于可以运行与 `xv6` 系统的可执行文件，我们会在学习了 `xv6` 的 `$(OBJS)` 内核主体代码之后，再来学习和分析这些 `xv6` 的应用程序。

我们用 `ll` 命令了解一下该磁盘文件系统的总容量，为 `500KB`。

```bash
➜  xv6-expansion git:(dev) ll fs.img   
-rw-rw-r-- 1 ubuntu ubuntu 500K Jan 23 13:14 fs.img
```

由于我们还没有学习 `xv6` 的文件系统格式，因此现在也无法分析 `fs.img` 的内容。之后我们会专门讨论 `mkfs` 工具的实现。

#### xv6mem.img 镜像

与 `xv6.img` 由启动扇区 `bootblock` 和内核 `kernel` 组成不同，`xv6mem.img` 还包含了完整的磁盘文件系统的内容（即 `kernelmemfs` = `kernel` + `fs.img`）。

```makefile
xv6memfs.img: bootblock kernelmemfs
    dd if=/dev/zero of=xv6memfs.img count=10000
    dd if=bootblock of=xv6memfs.img conv=notrunc
    dd if=kernelmemfs of=xv6memfs.img seek=1 conv=notrunc
```

### 2.4 xv6 的 Makefile

我们已经解读了 `xv6` 的 `Makefile` 的部分内容，随着分析和实验的开展，我们可能会解读和修改其中的部分内容，需要不时地查阅。

由于编译选项中加入了 `-MD` 选项，因此链接过程中分析依赖关系后，会输出 `.d` 文件。

各个应用程序的符号都通过 `.sym` 输出，例如 `wc.sym`。

其中还有 `Cuth`、`Dot-bochsrc` 等。

