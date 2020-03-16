---
title: 1. 磁盘裸设备的读写（实验）
---

该实验可以分为下面几个步骤：

1. 生成一个大小合适的文件，并写一些数据到文件中。
2. 将文件作为新的 IDE 磁盘添加到 xv6 中，设备号为 2。
3. 改造对应的 xv6 代码，尝试读取新磁盘中的数据，若读取成功则实验完成。

---

XV6 的文件系统中对磁盘的划分类似 EXT2 底层划分，从底层到调用，XV6 的文件系统分 6 层实现。

| 层   | 抽象接口 |
| ---- | -------- |
| 5    | 路径名   |
| 4    | 目录     |
| 3    | 索引节点 |
| 2    | 日志层   |
| 1    | 缓存层   |
| 0    | 磁盘     |

本节主要利用文件系统知识给 XV6 添加一个原生磁盘并实现简单的读写。

## 1. 生成裸磁盘

### 1.1 添加磁盘

首先我们查看 XV6 是如何添加磁盘的，先看看 `xv6.img` 的生成，打开 `Makefile`，相关语句如下：

```makefile
xv6.img: bootblock kernel
  dd if=/dev/zero of=xv6.img count=10000
  dd if=bootblock of=xv6.img conv=notrunc
  dd if=kernel of=xv6.img seek=1 conv=notrunc
```

可知 `xv6.img` 生成需要两个文件：`bootblock` 和 `kernel`。

第二行的 `dd` 指令中以 `/dev/zero` 为输入文件（input file），`xv6.img` 为输出文件（output file），拷贝 10000 块，一块默认为 512 字节，故生成的 `xv6.img` 总大小为 5120000 B，也就是 5000 KB。

第三行将 `bootblock` 的内容写入到 `xv6.img` 文件中。`conv` 是指定的参数转换文件（conversion），`notrunc` 表示不截短文件。

第四行将 `kernel` 文件复制到 `xv6.img` 中，`seek` 为偏移位，即从输出文件开头跳过 1 个块（512 字节）后再开始复制（第一个 512 字节给了 `bootblock`），同样是不截短文件的形式写入。

我们模仿 `xv6.img` 的生成，在 `Makefile` 中添加如下指令，手动生成一个 4096000 B 的空磁盘，每个盘块为 512 字节（8 个盘块等于 1 块物理页帧）。

```makefile
rawdisk.img:
  dd if=/dev/zero of=rawdisk.img bs=512 count=8000
  dd if=README.md of=rawdisk.img conv=notrunc
```

如果这里没有加参数 `notrunc`，那么读入 `README.md` 后，`rawdisk.img` 的大小将变为 `README.md` 的大小。添加后，命令行下执行如下命令，即可得到新磁盘信息，大小为。

```bash
# make     
# du -sh -b rawdisk.img
4096000	rawdisk.img
```

### 1.2 挂载磁盘

我们先看看 `xv6.img` 和 `fs.img` 这两个设备是如何挂载到操作系统上的，打开 `Makefile`，查找相关信息。可用看到 `Makefile` 的 QEMU 仿真命令选项 QEMUOPTS 将这两个文件挂载， 其中关于 `fs.img` 的命令如下

```bash
-drive file=fs.img,index=1,media=disk,format=raw
```

index 是设备号，media 是媒介类型，format 是格式。

QEMUOPTS 中有一个变量是 QEMUEXTRA，应该是用来扩展功能的，我们在 QEMUOPTS 的上面添加如下一行完成挂载。

```makefile
QEMUEXTRA = -drive file=rawdisk.img,index=2,media=disk,format=raw
```

在 `clean` 命令下添加 `rawdisk.img`，可以使用 `make clean` 清理生成的 `rawdisk.img` 文件。

## 2. 操作系统对磁盘的初始化

xv6 的 `main` 会调用 [ideinit()](https://github.com/professordeng/xv6-expansion/blob/dev/ide.c#L50) 对磁盘进行初始化。只有最后一个 CPU 打开了磁盘设备中断 `IRQ_IDE`。

## 3. 磁盘读写

缓存块 [buf](https://github.com/professordeng/xv6-expansion/blob/dev/buf.h) 的信息在 `buf.h`，磁盘的读写都依赖于缓存块。

磁盘读写函数是 [iderw()](https://github.com/professordeng/xv6-expansion/blob/dev/ide.c#L133)。

## 4. 提供系统调用读写磁盘

提供系统调用对 `rawdisk` 的读写，通过复用系统读超级块的代码，完成裸磁盘的读写。

## 5. 读写测试

### 5.1 原生系统功能

首先我们得学习一下 XV6 读文件的过程，然后再来进行模仿和创新。其中关于文件的函数有：

```c
open(name, perm);      // 打开或创建文件
write(fd, buf, len);   // 将 buf 中 len 个字节写入文件
close(fd)              // 关闭文件符，相当于进程不使用该文件
read(fd, buf, len);    // 将文件中的 len 字节读到 buf 中
unlink(name);          // 若硬链接为 1，unlink 操作表示删除文件
fstat(fd, stat);       // 将文件 fd 的状态信息读取到 stat 中
```

因此我们可用编写一个程序来读取 `README.md` 的数据以及文件信息，并显示出来。函数如下：

```c
char buf[512] = "hello world!";  // 全局变量

void readme() {
  int fd = open("README.md", O_RDONLY);   // fd >= 0 表示创建成功

  struct stat st;
  fstat(fd, &st);       // 读取文件信息 
  printf(1, "dev: %d\n", st.dev);
  printf(1, "ino: %d\n", st.ino);
  printf(1, "nlink: %d\n", st.nlink);
  printf(1, "type: %d\n", st.type);
  printf(1, "size: %d\n", st.size);

  read(fd, buf, 512);
  printf(1, "content is below:\n%s", buf);
  close(fd);
}
```

编写应用程序检验读写结果，检查 `rawdisk.img` 内容是否符合所写结果。为了忽略同步等繁琐的问题，我们在内核区定义一个全局变量，在 `main` 函数初始化时将磁盘的数据读到全局变量中，之后就可以用系统调用将数据读取出来。首先用 more 命令查看 `rawdisk.img` 的数据如下：

```bash
# more rawdisk.img
read rawdisk succeed!
```

