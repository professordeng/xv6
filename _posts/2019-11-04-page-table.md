---
title: 4. 分页机制
---

X86 处理器在进入保护模式之后，仅仅启用分段内存管理机制，分页机制还未开启。 如果启用分页硬件机制并结合软件的换页技术可以实现虚拟内存，从而让进程的编程空间可以大于系统的物理内存。OS 提供虚存管理后，使得多任务执行环境下的应用程序编程者对内存的使用更加简单和有效方便。 

由于 XV6 在启动时使用了 X86 硬件的大页模式，后续运行是采用 4 KB 页，因此读者需要对两种模式都有一些认识。 

64 位 `CR0` 寄存器的定义如下，当前关注的是 WP（Write Protect）位。

| Bits  | Mnemonic | Description            | R/W  |
| ----- | -------- | ---------------------- | ---- |
| 63~32 | Reserved | Reserved, Must be Zero |      |
| 31    | PG       | Paging                 | R/W  |
| 30    | CD       | Cache Disable          | R/W  |
| 29    | NW       | Not Writethrough       | R/W  |
| 28~19 | Reserved | Reserved               |      |
| 18    | AM       | Alignment Mask         | R/W  |
| 17    | Reserved | Reserved               |      |
| 16    | WP       | Write Protect          | R/W  |
| 15~6  | Reserved | Reserved               |      |
| 5     | NE       | Numeric Error          | R/W  |
| 4     | ET       | Extension Type         | R    |
| 3     | TS       | Task Switched          | R/W  |
| 2     | EM       | Emulation              | R/W  |
| 1     | MP       | Monitor Coprocessor    | R/W  |
| 0     | PE       | Protection Enabled     | R/W  |

## 1. 小页模式

在 32 位系统上，两级页表机制，全局页目录 PGD（Page Global Directory）和页目录 PMD （Page Middle Directory），保存数据的页和保存页表项的物理页在管理上没有什么本质区别。在 64 位系统上需要经过 4 级页表。 

页表内容由操作系统软件填写，而由地址硬件部件使用。页表虽由处理器地址部件使用，但却保存在 CPU 外部的内存上（近期常用的少量页表项保存在处理器内部的 TLB 中）。 每个进程有自己的全局页目录表 PGD，它通过进程控制块的成员 `p->pgdir` 指针来引用，实际上指向一个物理页帧。这个对应于 PGD 的页帧被看做一个类型为 `pgd_t` 类型的数组，例如 32 位 X86 中 4 KB 页帧包含数组元素有 1024 个，每个元素指向下一级页表（PMD）。进程切换时通过使用不同的 PGD 实现页表切换，从而让处理器看到相应的进程空间。X86 结构中是将切入进程的 `p->pgdir` 装入到处理器 CR3 寄存器来实现的。 

PGD 表中的每一项各自指向一个物理页帧，该页帧称为 PMD，包含多个 `pmd_t` 类型的页中间目录 PMD 表项。每个 PMD 项各自指向一个物理页帧，该页帧称为 PD，每个 PD 包含多个 `pte_t` 类型的页表项 PTE（Page Table Entries）。每个 PTE 项通常最终指向一个用于保存数据或代码的物理页帧，当 PTE 无效时需要额外信息指出从磁盘什么位置可以读到该页内容，文件映射页是通过相应 VMA 的 `vm_area_struct->vm_file` 指出所在磁盘文件，而匿名页换出后则在 PTE 中保存换出位置。于是，每一个线性地址都分成几个部分，分别是 PGD 索引号、PMD 索引号以及页内偏移。 

在 `XV6` 与页表处理相关的代码中，用 `pde_t` 表示一级页表的首地址，用 `pte_t` 统一表示页表的项，其实就是一个无符号整数。 

需要注意线性地址（Linear Address）指的是属于当前处理器上正在运行的进程空间，`p->pgdir` 给出的是物理地址，`pgd_t` / `pmd_t` / `pte_t` 项保存的地址也都是物理地址。运行时操作系统需要为进程设定 `p->pgdir`、将 `p->pgdir` 写入到 CR3 寄存器以及修改各级页表内容。处理器发出的每一个虚地址访问过程中的地址转换过程都是由硬件完成的。

下面以 X86 系统为例说明虚地址转换成物理地址的过程。在不支持 PAE（Physical Address Extension）的 32 位 X86 机器上使用两层页表。物理页大小为 4 KB，物理内存大小为 2 GB，CR3 寄存器指向第 48 号页帧，最终读入 `xyz` 的值。

1. [PDX](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L74) 宏用于取得虚地址的页目录索引值（31~22 位），然后从 PGD 中根据该索引找到 PMD 物理地址。
2. [PTX](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L77) 宏用于获得虚地址的页表索引值（21~12 位），然后从 PMD 中找到 PTE。
3. [PTE_PGADDR](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h#L100) 取 PTE 的高 20 位得到物理地址。

页表项结构如下表

| Bits  | Mnemonic | Description                   |
| ----- | -------- | ----------------------------- |
| 31~12 | PFN      | Physical Frame Number         |
| 11~9  | AVL      | Available for system use      |
| 8~7   | Reserved | Reserved                      |
| 6     | D        | Dirty（0 in page directory）  |
| 5     | A        | Accessed                      |
| 4     | CD       | Cache Disabled                |
| 3     | WT       | 1=Write-through, 0=Write-back |
| 2     | U        | User                          |
| 1     | W        | Writable                      |
| 0     | P        | Present                       |

页目录项和页表项唯一不同的就是 D 位。

## 2. 大页模式

XV6 一开始进入保护模式时使用大页模式，每个页有 4 MB，因此程序发出的虚地址（线性地址）被划分成 `10:22` 两个部分，前面 10 bit 用来做页目录索引，后面用来做页内偏移。因此 XV6 只需要使用一级页表既可以完成虚实地址转换，每个页表只需要占用 4 KB 即可。

在页表转换过程中，如果 PGD 项指出是大页，则不需要后续转换。

CR4 控制寄存器，定义如下表，我们现在只关心其中的 PSE 位。 

| Bits  | Mnemonic | Description                                    | Access Type |
| ----- | -------- | ---------------------------------------------- | ----------- |
| 63~19 | Reserved | Reserved                                       | Reserved    |
| 18    | OSXSAVE  | XSAVE and Processor Extended States Enable Bit | R/W         |
| 17~11 | Reserved | Reserved                                       | Reserved    |
| 10    | OSUES    | Operating System Unmasked Exception Support    | R/W         |
| 9     | OSFXSR   | Operating System FXSAVE/FXRSTOR Support        | R/W         |
| 8     | PCE      | Performance-Monitoring Counter Enable          | R/W         |
| 7     | PGE      | Page-Global Enable                             | R/W         |
| 6     | MCE      | Machine Check Enable                           | R/W         |
| 5     | PAE      | Physical Address Extension                     | R/W         |
| 4     | PSE      | Page Size Extensions                           | R/W         |
| 3     | DE       | Debugging Extensions                           | R/W         |
| 2     | TSD      | Time Stamp Disable                             | R/W         |
| 1     | PVI      | Protected-Mode Virtual Interrupts              | R/W         |
| 0     | VME      | Virtual-8086 Mode Extensions                   | R/W         |