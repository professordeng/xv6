---
title: 6. 调试 bootblock（二）
---

上节我们讲完 `bootasm.S` 的运行过程，无非就是两个功能

1. 从实模式进入到保护模式。实现 `i8086` 到 `i386` 的跳跃，即 16 位到 32 位的跳跃，主要完成 GDTR 的设置，以及相应的段寄存器的设置。
2. 调用 `bootmain`，完成 kernel 的装载过程。

由于 `bootasm.S` 的前期准备，`bootmain` 运行在保护模式下，其实也就是寻址范围变大了。由于 GDT 将段的起始地址均设置为 0，所以分段功能并没有完全实现，只不过多了相应的段保护，所以逻辑地址和线性地址都是一样的。

`bootmain` 的功能是将 kernel 从磁盘中找到内核程序，并将其调入内存，然后将控制器交给 kernel。

## 1. 读入 ELF 头部

内核 kernel 是以 ELF 格式的二进制文件的形式存在磁盘中，要读入内核，首先得知道关于内核的文件信息，信息一般在 ELF 文件的前 4096 字节，即 ELF 文件头。 我们先查看一下 ELF 文件头信息如下，有两个程序头。

```bash
# readelf -h kernel

ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x10000c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          178184 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         18
  Section header string table index: 17

```

一开始，`bootmain` 调用了 `readseg` 函数，目的是将 kernel 的 ELF 文件头从磁盘中读入内存，`readseg` 函数如下：

```c
// Read 'count' bytes at 'offset' from kernel into physical address 'pa'.
// Might copy more than asked.
void
readseg(uchar* pa, uint count, uint offset)
{
  uchar* epa;

  epa = pa + count;

  // Round down to sector boundary.
  pa -= offset % SECTSIZE;

  // Translate from bytes to sectors; kernel starts at sector 1.
  offset = (offset / SECTSIZE) + 1;

  // If this is too slow, we could read lots of sectors at a time.
  // We'd write more to memory than asked, but it doesn't matter --
  // we load in increasing order.
  for(; pa < epa; pa += SECTSIZE, offset++)
    readsect(pa, offset);
}
```

由于磁盘的读写是用偏移来表示的，所以 `offset` 表示磁盘文件的开始位置 0，`count` 表示文件的大小 4096 字节，`*pa` 指向物理内存的首地址 `0x10000`。盘块是一整块读取的，所以读取的数据可能大于 `count`。

由于 kernel 文件位于 `sector 1`，`sector 0` 用来存储 `bootblock`，`offset` 转化为盘块单位后需要加 1。

读取单个盘块的函数是 `readsect`，代码如下： 

```c
// Read a single sector at offset into dst.
void
readsect(void *dst, uint offset)
{
  // Issue command.
  waitdisk();
  outb(0x1F2, 1);   // count = 1
  outb(0x1F3, offset);
  outb(0x1F4, offset >> 8);
  outb(0x1F5, offset >> 16);
  outb(0x1F6, (offset >> 24) | 0xE0);
  outb(0x1F7, 0x20);  // cmd 0x20 - read sectors

  // Read data.
  waitdisk();
  insl(0x1F0, dst, SECTSIZE/4);
}
```

`readsect()` 里的函数均来源于 `x86.h` 里面封装的汇编函数。

ELF 文件头有固定的格式，用结构体 [elfhdr](https://github.com/professordeng/xv6-expansion/blob/master/elf.h#L5) 来表示。然后根据一个无符号整型变量来简单判断是否为 ELF 变量。这里的 [ELF_MAGIC](https://github.com/professordeng/xv6-expansion/blob/master/elf.h#L3) 是一个宏定义序列号。

```c
// Is this an ELF executable?
if(elf->magic != ELF_MAGIC)
  return;  // let bootasm.S handle error
```

然后载入相应的程序段头，段描述信息有 [proghdr](https://github.com/professordeng/xv6-expansion/blob/master/elf.h#L24) 结构体来表示。根据段描述信息调用 `readseg()` 将数据从磁盘中载入，并调用 `stosb()` 将段的剩余部分置零。[stosb()](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L42) 使用 x86 指令 `rep stosb` 来初始化内存块中的每个字节。

```c
// Load each program segment (ignores ph flags).
ph = (struct proghdr*)((uchar*)elf + elf->phoff);
eph = ph + elf->phnum;
for(; ph < eph; ph++){
  pa = (uchar*)ph->paddr;
  readseg(pa, ph->filesz, ph->off);
  if(ph->memsz > ph->filesz)
    stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
}
```

我们可以查看一下 kernel 的段信息如下，发现只有一个 LOAD 类型。

```bash
# readelf -l kernel             

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

载入完整的 ELF 文件后，将 ELF 可执行文件的入口地址赋给 [entry](https://github.com/professordeng/xv6-expansion/blob/master/entry.S#L44) 函数指针，紧接着就执行 `entry()` 函数，此时启动代码完成了引导作用，kernel 开始接管系统。

