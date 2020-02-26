---
title: 6. 文件相关系统调用
---

文件系统的用户态接口是在 [usys.S](https://github.com/professordeng/xv6-expansion/blob/master/usys.S) 中定义的，大部分在 [sysfile.c](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c) 中实现，也就是说用户的 C 程序中使用 `open()`、`close()`、`dup()`、`fstat()`、`read()`、`write()`、`mkdir()`、`mknod()`、`chdir()`、`link()`、`unlink()` 等形式的宏， 就可以进行相应的系统调用。

经过系统调用的公共入口之后，将分别转到 `sys_open()`、`sys_close()`、`sys_dup()`、`sys_fstat()`、`sys_read()`、`sys_write()`、`sys_mkdir()`、`sys_mknod()`、`sys_chdir()`、`sys_link()`、`sys_unlink()` 等具体系统调用服务函数中。

## 1. 文件打开与关闭

1. 文件描述符

   [fdalloc()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L38) 将扫描本进程的文件描述符数组 `proc->ofile[]`，如果其值为 0 则表示空闲可用，将它指向文件对象即可。

2. 文件对象

   `file.c` 文件中的 [filealloc()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L25) 用于分配文件对象，它将扫描系统的 `ftable.file[]`（共有 `NFILE` 个元素）数组，找到一个空闲的 `file` 对象然后将其引用计数 `ref` 增 1，表示被使用。

[sys_open()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L286) 是用于打开文件的通用操作，并不区分磁盘文件、设备文件或管道文件。这些文件的区别在文件读写操作的时候。

如果打开模式带有 `O_CREATE` 标志，则使用 `create()` 打开文件。

否则，先通过 `namei()` 查找相应的索引节点，然后由 `filealloc()` 分配文件对象，最后 `fdalloc()` 为该文件对象创建描述符。

## 2. 文件读写操作

[sys_read()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L69) 首先检查参数是否正确，然后调用 `fileread()` 完成读入操作。 

[sys_write()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L81) 首先检查参数是否正确，然后调用 `filewrite()` 完成读入操作。

## 3. 目录操作

[sys_mkdir()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L336) 在文件系统的目录树中创建一个目录节点，它是通过调用 [create()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L343) 创建相应的索引节点的，需要指定类型为目录 `T_DIR`。

[sys_mknod()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L352) 在文件系统的目录树中创建一个设备节点，共有三个参数：第一个参数是该节点所在的路径名，第二个参数是设备的主设备号 `major`，第三个参数是设备的次设备号 `minor`。具体操作的完成是通过 `create()` 完成的。

[sys_chdir()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L372) 将改变进程的当前目录，即 `proc->cwd`。它用 `argstr()` 从调用参数中获得新的路 径，然后借助 `namei()` 找到对应的索引节点。如果索引节点的类型是目录 `T_DIR`，则将本进程的工作目录 `proc->cwd` 设置为该索引节点。

## 4. 其他操作

[sys_pipe()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L423) 对应的用户态函数原型是 `user.h` 中的 `int pipe(int *)`，其中 `int *` 指向连续的两个整数，用做输入和输出两个文件描述符。`sys_pipe()` 内部主要就是通过 [pipealloc(&rf,&wf)](https://github.com/professordeng/xv6-expansion/blob/master/pipe.c#L22) 分配两个 `file` 结构体，然后为它们分配文件描述符 `fd0` 和 `fd1`，最后将这两个描述符返回给用户进程。

## 5. 代码总结

[sysfile.c](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c) 实现的文件系统接口，因此并不区分磁盘文件、管道或设备。另外文件的写操作部分，会和日志 [log](https://github.com/professordeng/xv6-expansion/blob/master/log.c) 代码有互动。文件系统层的抽象操作代码，向下将经过 `file` 层面代码，进入到 `inode` 层面的代码，然后根据文件类型不同而转向不同的处理函数：可能是管道、磁盘文件或设备的操作代码。

作为文件系统的最顶层，向上与应用进程交互，这需要通过文件描述符作为中介对象。每个进程维护着自己所打开的文件描述符列表，文件描述符只是对所打开文件的一个引用。[fdalloc()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L38) 在打开文件时用来获取本进程的一个空闲文件描述符，其过程很简单，就是扫描本进程的文件描述符数组 `proc->ofile[]`，找到未使用的一个，并将其指向文件的 `file` 结构体。