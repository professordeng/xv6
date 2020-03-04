---
title: 7. 多核启动
---

XV6 是个支持多核的操作系统。若是运行在多核处理器上，XV6 需要对 AP（non-boot CPU）进行初始化并启动。

XV6 启动时先将系统放入 BP（Boot Processor）中启动，BP 进入 `main()` 函数后首先进行一系列初始化，其中包括 `mpinit()`，此函数目的是检测 CPU 个数并将检测到的 CPU 存入一个全局的数组中。

BP 在 `startothers()` 中将 AP 启动后，进入忙等状态（死循环），AP 启动后会进入 `mpenter()` 函数，AP 在执行完一些初始化后最后也执行了 `mpmain()` 函数，并告诉 BP 继续往下执行。

每个 CPU 启动后都会执行 `mpmain()` 函数，在 `mpmain()` 中会打印输出当前正在启动的 CPU ID，然后初始化 IDT，将 CPU 已启动标志置 1，最后开始进程调度。若有任何一个 AP 的启动标志没有开启，BP 就不会往下执行，所以 BP 是最后进入 `mpmain()` 函数中的。 

## 1. 检测多核信息

BP 在执行 `main` 函数时，会调用 [mpinit](https://github.com/professordeng/xv6-expansion/blob/master/mp.c) 来检测其他核的信息。在 `mp.c` 中定义了 3 个全局变量，分别是 `cpus[]`、`ncpu`、`ioapicid`。`mpinit()` 就是初始化这些变量的。

## 2. 启动 AP

BP 在 `startothers()` 中利用 [lapicstartap](https://github.com/professordeng/xv6-expansion/blob/master/main.c#L89) 指令让 AP 启动，AP 会跳到 `entryother.S` 开始执行，AP 的启动和 BP 类似。









