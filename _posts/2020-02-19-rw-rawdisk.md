---
title: 2. 磁盘裸设备的读写
date: 2019-11-03
---

XV6 的文件系统中对磁盘的划分类似 EXT2 底层划分，从底层到调用，XV6 的文件系统分 6 层实现。本文主要利用文件系统知识给 XV6 添加一个原生磁盘并实现简单的读写。

## 1. 生成裸磁盘

首先我们查看 XV6 是如何添加磁盘的，先看看 `xv6.img` 的生成，打开 `Makefile`，相关语句如下：

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

我们模仿 `xv6.img` 的生成，手动生成一个 512000 B 的空磁盘，每个盘块为 512 字节。

```bash
dd if=/dev/zero of=rawdisk.img bs=512 count=1000
du -sh -b rawdisk.img   # 查看磁盘大小
```

此时目录下会多一个名为 `rawdisk.img` 的文件，这个文件将作为新的磁盘。我们可以搭建自己的文件系统。

## 2. 给操作系统添加新设备

我们先看看 `xv6.img` 和 `fs.img` 这两个设备是如何添加进操作系统的，打开 `Makefile`，查找相关信息。可用看到 `Makefile` 的 `qemu` 仿真命令选项 `QEMUOPTS` 将这两个文件添加了进入， 其中关于 `fs.img` 的命令如下

```bash
-drive file=fs.img,index=1,media=disk,format=raw
```

`index` 是设备号，media 是媒介类型，format 是格式。我们依照此格式将 `rawdisk.img` 添加到 XV6，将下面语句添加到 `QEMUOPTS` 选项。 

```bash
-driver file=rawdisk.img,index=2,media=disk,format=raw
```

## 3. 操作系统对磁盘的初始化

系统启动时需要对 `rawdisk` 盘做一定的初始化，然后才能调用相应的函数访问。这里可以自己初始化布局，也可以参照 XV6 的磁盘布局。

## 4. 提供系统调用读写磁盘

提供系统调用对 `rawdisk` 的读写

## 5. 测试

编写应用程序检验读写结果，检查 `rawdisk.img` 内容是否符合所写结果。

可以利用 `dd` 指令直接读取。