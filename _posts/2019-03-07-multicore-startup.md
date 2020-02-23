---
title: 7. 多核启动
---

XV6 是个支持多核的操作系统。实现代码集中在 [mp.h](https://github.com/professordeng/xv6-expansion/blob/master/mp.h) 和 [mp.c](https://github.com/professordeng/xv6-expansion/blob/master/mp.c)。

## 1. 多核启动过程

XV6 启动时先将系统放入 BSP（Bootstrap Processor）中启动，BSP 进入 `main()` 函数后首先进行一系列初始化，其中包括 `mpinit()`，此函数目的是检测 CPU 个数并将检测到的 CPU 存入一个全局的数组中，之后进入 `startothers()` 函数通过向 AP（non-boot CPU）发送中断的方式来启动 AP，最后执行 `mpmain()` 函数。

BSP 在 `startothers()` 中将 AP 启动，启动后会进入 `mpenter()` 函数，AP 在执行完一些初始化后最后也执行了 `mpmain()` 函数。

每个 CPU 启动后都会执行 `mpmain()` 函数，在 `mpmain()` 中会打印输出当前正在启动的 CPU ID，然后初始化 IDT，将 CPU 已启动标志置 1，最后开始进程调度。至此，多核启动完成。

## 2. 检测多核的方法

系统首先查找 MP（multi-processor）浮点结构，步骤如下：

1. 如果 BIOS 扩展资料区域（EBDA）已经定义，则在其中的第 1 KB 中进行查找。
2. 若 EBDA 未被定义，则在系统基本内存的最后 1 KB 中寻找。
3. 在 BIOS ROM 里的 `0xF0000` 到 `0xFFFFF` 的地址空间中寻找。

