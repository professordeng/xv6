---
title: 2. 物理内存
date: 2019-05-03
---

我们下面默认虚拟空间的页称为虚页，物理空间的页称为页帧。

物理内存的初始化分成两步：

1. 大叶模式的早期布局
2. 启动 4 KB 分页后的空闲物理页帧初始化。目的是为了摸清物理内存资源总量，并通过链表管理所有空闲页帧。 

## 1. 早期布局

在 `kernel main()` 函数最开始处，`xv6` 启用大页模式并具有以下的内存布局：内核代码存在于物理地址低地址的 `0x100000` 处，而虚存空间的 `0~0x0040-0000` 和 `0x8000-0000~0x8040-0000` 地址开始的两段 4 MB 都影射了内核代码。此时的早期页表为 `main.c` 文件中的 `entrypgdir` 数组，其中只有两项有效：

1. 虚拟地址低 `[0, 4MB]` 映射物理地址低 `[0, 4MB]`。
2. 虚拟地址 `[KERNBASE, KERNBASE+4MB)` 映射到物理地址 `[0, 4MB]`。

此时只使用了 一个物理页（4 MB 的大页）并映射到两个不同的虚存空间。 

## 2. 物理页帧的初始化

可见此时 kernel 实际能用的虚拟地址空间显然是不足以完成正常工作的，所以初始化过程中需要获得更多可用物理页帧并重新设置页表。

### 2.1 空闲物理页帧

启用 4 KB 分页之后，物理内存划分成 4 KB 大小的页帧来管理，空闲的物理页帧构成一个 链表，页帧开头的 4 个字节用来做指针，形成空闲物理页帧链表。 

由于 `xv6` 没有实现对 `x86` 中确定内存总量的测定，因此只是简单地使用总量为 240 MB 的物理内存（`PHYSTOP`），因此在内核刚启动时，从 kernel 结束地址 end 直到 240 MB 的空间都是空闲的。 

`xv6` 在 `main()` 函数中调用 `kinit1()` 和 `kinit2()` 来初始化物理内存，将空闲物理页帧构成链表。 需要注意的是除了启动时使用了 4 MB 的大页模式外，`xv6` 正常运行时使用的是 4 KB 页。其中 `kinit1()` 初始化第一个 4 MB 的物理范围，其中从 kernel 结束处到 4 MB 边界的物理内存空间组织为未使用（虽然此时为 4 MB 页，但是按照 4 KB 页帧进行组织），`kinit2()` 初始化剩余内核空间到 `PHYSTOP` 为未使用（`kinit2()` 在 4 KB 页的环境下工作，前面的 `main()->kmalloc()` 完成 4 KB 页表的建立）。相关的具体代码分析请见 [kalloc.c](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c)。 

上述两个函数的核心是 [kalloc.c#L46](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c#L46) 的 `freerange()`，该函数将传入的地址范围 `(vstart， vend)` 之间的所有页，逐个通过 `kfree()` 登记为空闲页。[kalloc.c#L59](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c#L59) 的 `kfree()` 将一个页插 入到空闲页帧链表 `kmem.freelist` 的头部。 

因此当 `main()->kinit1(end, P2V(4*1024*1024))` 执行结束时，`(end,4MB)` 区间的物理页帧 将构成一个单向链表，表头为 `kmem.freelist`。而 `kinit2()` 则把 `(4MB，PHYSTOP)` 范围内的物理页帧插入到空闲链中。如下表

| 0~end          | end~4 MB                  | 4 MB~PHYSTOP（240 MB）    |
| -------------- | ------------------------- | ------------------------- |
| 内核占用的内存 | `kinit1()` 收集的空闲内存 | `kinit2()` 收集的空闲内存 |

### 2.2 kalloc.c 和 mmu.h

[kalloc.c](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c) 中有刚讨论过的物理内存初始化的 `kinit1()` 和 `kinit2()`，以及页帧分配 `kalloc()` 和回收 `kfree()` 等函数。[mmu.h](https://github.com/professordeng/xv6-expansion/blob/master/mmu.h) 中则有大量关于页表映射相关的常量、宏和函数。

[kalloc.c#L16](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c#L16) 行定义了 run 结构体用于形成页帧链表，[kalloc.c#L20](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c#L20) 定义了 `kmem` 结构体用于管理空闲物理页帧链表。`kmem` 成员变量包括空闲页帧链表指针 `freelist` 以及互斥锁 `lock`。

`kinit1()` 和 `kinit2()` 也定义在这里，两者实现非常相似，差别在于 `kinit1()` 还对互斥锁进行初始化，以及两者处理的地址范围也不同。`kinit1()` 和 `kinit2()` 都利用 `freerange()` 将 `vstart~vend` 地址范围内的页帧添加到 `freelist` 链表中。当初始化完成之后，`freerange()` 用于回收页帧（当然， 也可以将初始化看成某种意义上的回收操作）。`freerange()` 对 `vstart~vend` 所覆盖的物理页帧， 逐个调用 `kfree()` 进行回收，每次一个页帧。

在系统正常运行时，需要分配页帧时将调用 `kmallc()`，它将扫描 `kmem.freelist` 链表，从中摘取一个页帧并返回。

## 3. 页帧的分配与回收

`kalloc.c` 中的代码一部分参与物理内存管理子系统的初始化；另一部分则是关于内存分配与回收操作，主要涉及分配函数 `kalloc()` 和回收 `kfree()`。我们现在来主要分析分配与回收，这两 个函数都定义与 `kalloc.c`。 `kfree()` 对地址做一些合法性检查，然后 [kalloc.c#L72](https://github.com/professordeng/xv6-expansion/blob/master/kalloc.c#L72) 将该页插入到队列头部。而 `kalloc()` 则是在空闲链表头部取下一个页帧来完成分配操作。`kalloc()` 返回虚拟地址空间的地址，`kfree()` 以虚拟地址为参数，通过 `kalloc()` 和 `kfree()` 能够有效管理物理内存，让上层只需要考虑虚拟地址空间。 由此可见 `xv6` 对物理页帧的管理非常简单，并没有考虑系统中的其他因素，也不需要支持虚拟内存的换出操作。 

