---
title: 2. 磁盘裸设备的读写
date: 2019-11-03
---

c本节学习如何给 `xv6` 添加一个额外的磁盘，在实验的过程中加深 `xv6` 文件系统的理解。

## 1. 生成裸磁盘

首先我们查看 `xv6` 是如何添加磁盘的，先看看 `xv6.img` 的生成，打开 `Makefile`，相关语句如下：

```makefile
xv6.img: bootblock kernel
  dd if=/dev/zero of=xv6.img count=10000
  dd if=bootblock of=xv6.img conv=notrunc
  dd if=kernel of=xv6.img seek=1 conv=notrunc
```

`xv6.img` 生成需要两个文件：`bootblock` 和 `kernel`。

第二行的 `dd` 指令中以 `/dev/zero` 为输入文件（input file），`xv6.img` 为输出文件（output file），拷贝 10000 块，一块默认为 512 字节，故生成的 `xv6.img` 总大小为 5120000 B，也就是 5000 KB。

第三行将 `bootblock` 的内容写入到 `xv6.img` 文件中。`conv` 是指定的参数转换文件（conversion），`notrunc` 表示不截短文件。

第四行将 `kernel` 文件复制到 `xv6.img` 中，`seek` 为偏移位，即从输出文件开头跳过 1 个块（512 字节）后再开始复制（第一个 512 字节给了 `bootblock`），同样是不截短文件的形式写入。

**生成裸磁盘文件**

我们模仿 `xv6.img` 的生成，手动生成一个 512 000 B 的空磁盘，每个盘块为 512 字节。

```bash
dd if=/dev/zero of=rawdisk.img bs=512 count=1000
du -sh -b rawdisk.img
```

此时目录下会多一个名为 `rawdisk.img` 的文件，这个文件将作为新的磁盘。我们可以搭建自己的文件系统。

## 2. 给操作系统添加新设备

我们先看看 `xv6.img` 和 `fs.img` 这两个设备是如何添加进操作系统的，打开 `Makefile`，查找相关信息。

2. 修改 `Makefile` 的 `qemu` 仿真命令选项 `QEMUOPTS`， 加入：

   ```bash
   -driver file=rawdisk.img,index=2,media=disk,format=raw
   ```

   `index` 是新增磁盘的设备号

3. 系统启动时对 `rawdisk` 盘的初始化

4. 提供系统调用对 `rawdisk` 的读写

5. 编写应用程序检验读写结果，检查 `rawdisk.img` 内容是否符合所写结果。

