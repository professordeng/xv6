---
title: 7. 代码与数据隔离
date: 2019-08-13
---

`xv6` 操作系统是一个非常简陋的原型系统，因此并未没有对进程影像中的代码和数据进行区分，更没有对它们进行访问权限控制。下面我们来看看 `xv6` 进程可以如何破坏自己的代码和数据，然后再分析如何将代码和数据按不同属性进行区分管理。

 ## 1. 破坏程序数据

我们编写 `no_protect.c` 应用程序，然后展示它如何破坏自己的数据和代码。记得在 `Makefile` 中添加 `_no_protect\`。

```c
#include "types.h"
#include "stat.h"
#include "user.h"

int
main()
{
  char *p = (char*)0x0000;
  for(int i = 0x0000; i < 0x08; i ++){
    *(p+i) = '*';
  }
  printf(1, "This string shouldn't be modified!\n");
  exit();
}
```

编译后，用 `readelf -l _no_protect` 查看可执行文件 `_no_protect` 的信息，如下：

```bash
Elf file type is EXEC (Executable file)
Entry point 0x0
There are 2 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000080 0x00000000 0x00000000 0x00a28 0x00a34 RWE 0x10
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .eh_frame .bss 
   01   
```

从中可以看出，整个可执行文件只有一个类型为 `LOAD` 的段需要装入内存，具体是从磁盘文件 `0x000080` 地址开始装入 `0x00a28` 字节到内存 `0x000000` 地址范围，并占用 `0x000000~0x000a34` 的范围。

### 1.1 对进程影像的破坏

之前提到过 `xv6` 可执行文件的生成中，使用了 `-N` 链接选项，使得 `-N` 参数用于指出 `data` 节与 `text` 节都是可读可写、不需要在页边界对齐。也就是说 `xv6` 可执行文件只形成一 个段，其属性为 `RWE`（即可以读、写、执行），完全没有进行权限控制，可以修改任何 一个字节。 

虽然在 `xv6` 系统中运行 `no_protect` 后是完全正常的，如下：

```markdown
$ no_protect
This string shouldn't be modified!
```

但实际上代码的第 9~10 行的循环代码修改了代码的前 8 个字节，只不过这 8 字节代码在循环开始前已经被执行过了， 因此对程序行为并未造成影响。 

但是如果将写入长度扩展到 `0x0100`，将覆盖程序代码从而引起错误，如下： 

```bash
$ no_protect
pid 3 no_protect: trap 14 err 4 on cpu 1 eip 0x1a addr 0x3dc168--kill proc
```

同理，将写入地址从 `0x0` 移动到 `0x1000` 开始，即写入 `0x1000~0x1008` 也会引起内存访问错误，如下。因为该区域是 `xv6` 进程代码和堆栈之间的一个隔离区，该页未建立页表映射。

```bash
$ no_protect
pid 4 no_protect: trap 14 err 7 on cpu 1 eip 0x11 addr 0x1000--kill proc
```

但是将地址修改到 `0x2000` 开始的 8 字节，则又恢复正常。这是因为该地址区间是该进程的堆栈区。再继续将地址上移到 `0x3000` 将访问到未映射区，出现类似 `0x1000` 的错误（提示的地址将变成 0x3000）。 

除了破坏代码、访问未分配的内存区间外，还可能会修改只读数据且不引起程序致命错误。例如我们可以将指针指向所打印的字符串，从而影响后面打印输出结果。首先用 `readelf -S _no_prtect` 查看节的信息，可以看到有只读数据 `.rodata` 节，如下

```bash
There are 16 section headers, starting at offset 0x2f10:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000080 000756 00 WAX  0   0 16
  [ 2] .rodata           PROGBITS        00000758 0007d8 00003d 00   A  0   0  4
  [ 3] .eh_frame         PROGBITS        00000798 000818 000290 00   A  0   0  4
  [ 4] .bss              NOBITS          00000a28 000aa8 00000c 00  WA  0   0  4
  [ 5] .comment          PROGBITS        00000000 000aa8 00002b 01  MS  0   0  1
  [ 6] .debug_aranges    PROGBITS        00000000 000ad8 0000a0 00      0   0  8
  [ 7] .debug_info       PROGBITS        00000000 000b78 0009cd 00      0   0  1
  [ 8] .debug_abbrev     PROGBITS        00000000 001545 0004ff 00      0   0  1
  [ 9] .debug_line       PROGBITS        00000000 001a44 000309 00      0   0  1
  [10] .debug_str        PROGBITS        00000000 001d4d 000218 01  MS  0   0  1
  [11] .debug_loc        PROGBITS        00000000 001f65 0009d0 00      0   0  1
  [12] .debug_ranges     PROGBITS        00000000 002935 000080 00      0   0  1
  [13] .symtab           SYMTAB          00000000 0029b8 0003a0 10     14  21  4
  [14] .strtab           STRTAB          00000000 002d58 00011d 00      0   0  1
  [15] .shstrtab         STRTAB          00000000 002e75 00009a 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)

```

然后用 `objdump -s -j .rodata _no_protect` 查看其内容，如下

```bash
_no_protect:     file format elf32-i386

Contents of section .rodata:
 0758 54686973 20737472 696e6720 73686f75  This string shou
 0768 6c646e27 74206265 206d6f64 69666965  ldn't be modifie
 0778 64210a00 286e756c 6c290000 30313233  d!..(null)..0123
 0788 34353637 38394142 43444546 00        456789ABCDEF.  
```

我们修改程序第 8 行代码将 p 指向 0x0758 地址，这样将会把 `This string shouldn't be modified!\n` 的前 8 个字符修改为 `*`，运行结果如下：

```markdown
$ no_protect
********ing shouldn't be modified!
```

## 2. 代码与数据的隔离

为了对代码和数据进行保护，首先就需要按访问属性的不同进行分别管理。进程的数据和代码要分离，我们可以利用链接器命令或链接器脚本（`ld` 脚本）上做控制。我们将 `Makefile` 中关于可执行文件生成的规则进行修改，删除了 `-N` 选项。 

```makefile
_%: %.o $(ULIB)
  $(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $@ $^
  $(OBJDUMP) -S $@ > $*.asm
  $(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym
```

将链接命令中的 `-N` 选项去掉后，可以生成磁盘可执行文件 `_no_protect` 时，是有独立代码段和独立数据段，如下所示。读者可以回顾之前一个 `LOAD` 类型的段的情况。

```bash
Elf file type is EXEC (Executable file)
Entry point 0x0
There are 3 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x001000 0x00000000 0x00000000 0x00a28 0x00a28 R E 0x1000
  LOAD           0x002000 0x00002000 0x00002000 0x00000 0x0000c RW  0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .eh_frame 
   01     .bss 
   02  
```

修改之后依然可以重启 `xv6`，因为本人 `GCC` 版本是 `7.4.0`，可以通过 `gcc --version` 查看版本，如果是 `5.4.0`，则会因为 ELF 可执行文件段的起始地址不在页边界上而出现运行不了的问题，这涉及 `exec()` 和 `loaduvm()` 上有关页边界对齐的问题。

### 2.1 访问权限控制

将可执行文件的代码和数据地址范围分离后，还需要进一步修改它们的访问属性，将代码段修改为不可写。需要修改 `allocuvm()` 函数中的 `mappages()` 所使用的访问属性。这些工作等我有时间再补齐。

 