---
title: 5. 特殊文件
date: 2019-09-09
---

## 1. 设备文件

## 2. 管道文件

管道的本质是内核中的 一段内存，可以通过文件系统的接口来访问。[pipe.c#L13](https://github.com/professordeng/xv6-expansion/blob/master/pipe.c#L13) 定义的 pipe 结构体中，最核心的是一个数据缓冲区 `char data[PIPESIZE]`（`PIPESIZE=512`），使用自旋锁 `lock` 成员变量来保护，`nread` 和 `nwrite` 分别记录已经读和写的字节数，`readopen` 和 `writeopen` 用于记录该管道是否仍有读者或写者。 

