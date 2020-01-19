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

`bootblock` 的主要任务就是将 `xv6` 内核 `kernel` 文件读入，并将控制权转交给 `kernel` 代码，从而运行 `xv6` 代码，从而运行 `xv6` 操作系统。`kernel` 代码本身又包括主体代码和辅助初始化代码两部分。

-  kernel 的生成

  `xv6` 内核主体代码由许多独立的 C 源程序编译、链接而成，其中主要的源代码用于实现操作系统的进程管理、内存管理、文件系统和设备等，再加上辅助的初始代码等。

  `entry.S` 代码负责从 `x86` 的线性地址模式转向分页的虚地址模式、建立页表映射等内核启动的初始化工作。而 `$(OBJS)` 所对应的 `xv6` 内核主体就是运行在分页地址模式之下，使用页表映射的虚拟内存。

- 编译

  查看 `Makefile` 脚本。

  ```bash
  wc -l kernel.sym    # 统计 kernel.sym 符号个数
  head kernel.sym     # 查看 kernel.sym 的前十行
  ```

- 链接

  源码有一个链接脚本 `kernel.ld` ，在 `Makefile` 中该脚本将 `entry.o` 和 `$(OBJS)` 进行链接，参数 `-b binary` 之后的 `initcode` 和 `entryother` 按照二进制方式安排布局在指定地址。也就是说 `initcode` 和 `entryother` 是已经经过链接布局的，并不需要像 `$(OBJS)` 和 `entry.o` 那样（会将同类的 `.text .data .bss` 进行合并布局），而是安装自己原有的 ELF 格式进行独立布局，并拼接在 `entry.o` 和 `$(OBJS)` 链接结果的后面。
  
  观察 `kernel.ld` 脚本。发现其内核代码的地址布局从 `0x80100000` 开始的，但是其代码的装载地址是 `0x100000` 。后面解释装载地址和运行地址差异。
  
  `ENTRY(_start)` 中 `_start` 表示入口地址，在 `entry.S` 中定义取值为 `0x0010000c` 记录在 ELF 文件的文件头结构体 `elfhdr` 的 `entry` 成员中，即 `elfhdr.entry` 。
  
  用 `readelf -l kernel` 查看 kernel 。可以看到只有 00 段需要载入， `FielSiz` 刚好是 `0x0a516` 正好是 `entryother` 结束的地址。

### 3. 辅助初始化代码

kernel 文件中包含了一部分代码用于 `init` 进程的初始化、其他处理器的启动初始化的代码。其中 `init`  进程的代码将被装载到 `0x000` 地址，而其他核启动初始代码将装入 `0x7000` 地址（和链接时的地址一致）给其他处理器作为启动代码。具体查看 `Makefile` 注释。

### 4.kernelmemfs 的生成

当使用内存盘的时候，对应的内核不是 kernel 文件，而是 `kernelmemfs` 。该内核将完整的磁盘文件系统影像包含进来。当启动扇区装载这个 `kernelmemfs` 内核的时候，已经把整个文件系统一起装进内存，因此后续的磁盘操作都按内存操作方式来完成。`kernelmemfs` 的生成方式查看 `Makefile` 。类似 kernel 。

1. 编译

   当用 `kernelmemfs` 镜像的时候，内核的主体代码不是由 `$(OBJS)` 构成的，而是用 `$(MEMFSOBJS)` 弃掉 `ide.o` 加上 `memide.o` 构成。

2. 链接

   `kernelmemfs` 的链接和 `kernel` 类似，也是使用 `kernel.ld` 链接脚本，因此具有相同的内存和布局。不同之处在于 `-b binary` 后面多了个 `fs.mg` 。即磁盘镜像和内核链接成一个单一的 ELF 文件。其中 `fs.img` 是磁盘文件系统的影像，`fs.img` 稍后分析。

   执行 `make kernelmemfs` ，然后比较 `kernelmemfs` 和 `kernel` 的大小，发现 `kernelmemfs` 远大于 `kernel` 文件，和我们预想一样。

   接着我们通过`readelf -l kernelmemfs`查看 `kernelmemfs` 的段和节的情况，可以看出需要 LOAD 装入的 00 段比较大（对比 kernel 的 `0x0a526`）

   但是查看 `kernelmemfs` 的节，从 `.data` 节的大小我们可以知道 `fs.img` 是以数据的形式并入到 `.data` 节（Size 为 `07f516`）。

   再进一步，用 `readelf -s kernelmemfs | grep fs.img` 查看符号表，发现在数据节（5 号节）有 `_binary_fs_img_start` ，这是连接器以 `binary` 方式将 `fs.img` 链接进来的时候创建的符号。其地址位于 `0x8010a516 `，正好是在原来 `kernel` 结束的地方。（kernel 大小为 `0x516`），另外 `fs.img` 结束符号 `_binary_fs_end=80187516` 。链接器也为 `fs.img` 的大小创建了符号 ` ABS _binary_fs_img_size=0007d000` ，正好等于 `fs.img` 的大小 `0x7d000 = 51200` 。

### 4. 磁盘镜像

在内核启动之后，需要有文件系统保持程序和数据，例如 shell 中所执行的外部指令：`ls、mkdir、rm、cp` 等程序，以及用户所需的其他数据文件等。由于我们将 `xv6` 运行于 `QEMU` 仿真环境，因此需要把相应的文件系统内容形成磁盘影像文件。

 