---
title: kernelmemfs
---

虽然大多数情况下 XV6 使用 IDE 磁盘作为文件系统载体，但是某些情况下也会使用内存盘作为文件系统的载体。当使用内存盘的时候，对应的内核不是 `kernel` 文件，而是 `kernelmemfs` 。该内核将完整的磁盘文件系统影像包含进来。当启动扇区装载这个 `kernelmemfs` 内核的时候，已经把整个文件系统一起装进内存，因此后续的磁盘操作都按内存操作方式来完成。`kernelmemfs` 的生成过程如下所示，与 kernel 的生成过程类似。

1. kernel 主体部分

   `pro.o`、`fs.o`、`vm.o` 和 `memide.o` 组成的变量 `$(MEMFSOBJS)` 、`entry.o` 、`initcode` 和 `entryother`。

2. `fs.img` 组成内存中的磁盘文件系统。

下面讲解形成 `kernelmemfs` 的具体细节。

## 1. 编译

当用 `kernelmemfs` 影像时，内核的主体代码不是由 `$(OBJS)` 构成的，而是用 `$(MEMFSOBJS)`（将 `$(OBJS)` 中的 `ide.o` 替换为 `memide.o`）构成的。因为此时磁盘文件系统是使用内存模拟的，因此文件系统读写磁盘时不再使用 `ide.o` 中的代码，而是利用 `memide.o` 中的代码。

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

## 2. 链接

`kernelmemfs` 的链接和 `kernel` 类似，也是使用 `kernel.ld` 链接脚本，因此具有相同的内存和布局。不同之处在于 `-b binary` 后面多了个 `fs.img` （即磁盘影像和内核链接成一个单一的 ELF 文件。其中 `fs.img` 是磁盘文件系统的影像（`fs.img` 的生成过程会在后面分析）。

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

再进一步，用 `readelf -s kernelmemfs | grep fs.img` 查看 `kernelmemfs` 的符号表，发现在数据节（5 号节）有 `_binary_fs_img_start` ，这是链接器以 `binary` 方式将 `fs.img` 链接进来的时候创建的符号。其地址位于 `0x8010a516 `，正好是在原来 `kernel` 结束的地方。（kernel 大小为 `0x516`），另外 `fs.img` 的结束位置符号 `_binary_fs_end=80187516` 。链接器也为 `fs.img` 的大小创建了符号 ` ABS _binary_fs_img_size=0007d000` ，正好等于 `fs.img` 的大小 `0x7d000 = 512000` 。

```bash
121: 8010a516     0 NOTYPE  GLOBAL DEFAULT    5 _binary_fs_img_start
255: 0007d000     0 NOTYPE  GLOBAL DEFAULT  ABS _binary_fs_img_size
460: 80187516     0 NOTYPE  GLOBAL DEFAULT    5 _binary_fs_img_end
```

