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

## 1. bootasm.S

