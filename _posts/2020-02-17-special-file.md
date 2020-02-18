---
title: 5. 特殊文件
date: 2019-09-09
---

## 1. 设备文件

## 2. 管道文件

管道的本质是内核中的 一段内存，可以通过文件系统的接口来访问。[pipe.c#L13](https://github.com/professordeng/xv6-expansion/blob/master/pipe.c#L13) 定义的 pipe 结构体中，最核心的是一个数据缓冲区 `char data[PIPESIZE]`（`PIPESIZE=512`），使用自旋锁 `lock` 成员变量来保护，`nread` 和 `nwrite` 分别记录已经读和写的字节数，`readopen` 和 `writeopen` 用于记录该管道是否仍有读者或写者。 

**访问接口**

创建管道的用户态函数是 `pipe(int*)`，对应的系统调用是 `sys_pipe()`，最终实现是 `pipe.c` 中 的 [pipealloc()](https://github.com/professordeng/xv6-expansion/blob/master/pipe.c#L22)。

管道一旦被创建之后，后续访问是通过文件系统的系统调用来完成的，甚至管道的撤销也是通过 `close()` 完成的。因此管道相关的代码只需负责创建管道对象，以及将管道文件类型设置为 `FD_PIPE` 即可。

### 2.1 相关代码

大部分实现代码在 [pipe.c](https://github.com/professordeng/xv6-expansion/blob/master/pipe.c)。 

管道创建使用 `pipealloc()`，主要工作是分配 `pipe` 结构体，并创建两个 `file` 结构体分别用于读端和写端。为了和文件系统的 `read()`、`write()` 和 `close()` 对接，`xv6` 为管道提供了 `piperead()`、`pipewrite()` 和 `pipeclose()`。 