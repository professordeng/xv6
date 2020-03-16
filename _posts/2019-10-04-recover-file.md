---
title: 4. 恢复被删除的文件（实验）
---

若想恢复被删除的文件，至少得满足以下两个条件。

1. 执行 `rm` 操作后，`inode` 中的索引图并没有被清除，因为如果 `inode` 索引图被清除了，那么就找不到对应的数据盘块了。
2. 数据盘块里面的数据没有被擦除，由于 xv6 删除操作仅仅对位图进行更新，所以只要数据盘块没有被重写，数据就没有被擦除。

所以我们要对条件 1 进行相应的实现。在 xv6 中，若 `inode` 中的 `ref` 和 `nlink` 都为 0 时，`inode` 就会被擦除。`ref` 表示进程的引用次数，`nlink` 表示有多少个文件名。这里我们让 `ref` 不为 0 即可。

## 1. 细节

xv6 中文件是由 [inode](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L98) 标识，在 IO 层实现中，会调用 [bfree()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L80) 更新位图释放文件所占用的数据盘块（将位图位置 0），并没有磁盘中的数据删除（直到被重写）。一个盘块由二元组（dev, off）决定，所以要恢复被删除的文件，就要找到存储文件的所有的盘块，并按顺序组织起来。

无名文件由磁盘上的 `dinode` 唯一表示，`dinode` 在内存中的表现形式为 `inode`，我们统称索引节点。

`dinode` 结构包括：类型、大小、链接数、数据盘块的索引表。

所有索引节点连续地存放在磁盘上，起始于 `sb.startinode`。每个 `inode` 有一个号码，指明其 `dinode` 在磁盘中的位置。

内核维护着一个由 `inode` 组成的表，`inode` 比 `dinode` 多了不少信息，包括 `ref` 和 `valid`，`ref` 记录被进程引用的次数。

 索引节点被文件系统调用之前要经过一系列的状态变化。包括

1. 已分配状态：一个 `dinode` 的 `type` 非零。`ialloc()` 分配索引节点，若 `ref` 和 `nlink` 为零 `iput()` 释放索引节点。
2. 被引用状态：`ref` 为零表示空闲，否则 `ref` 记录指向 `inode` 的指针个数（文件描述符或目录），`iget()` 寻找或创建一个 `inode` 并使 `ref++`，`iput()` 使 `ref--`。
3. 有效状态：只有 `ip->valid` 为 1 时，`(type, size, &c)` 才有效，`ilock()` 从磁盘中读取 `dinode` 并设置 `valid` 为 1。在 `iput()` 中若 `ref` 为零，`valid` 被设置为 0。
4. 加锁状态：在加锁状态下，文件系统的代码才能修改 `inode` 的信息。

`iget()` 和 `ilock()` 分开使用是因为进程调用 `iget()` 可以长时间引用着 `inode`（但不使用），直到需要修改 `inode` 的时候才使用 `ilock()` 加锁

##  2. 删除过程

要想恢复文件，就得知道 xv6 是如何删除文件的。xv6 已经实现了Linux 的 `rm` 命令，实现源码在 [rm.c](https://github.com/professordeng/xv6-expansion/blob/master/rm.c) 中。

代码及其简洁，这里调用了系统调用 `unlink`，可以发现 `rm` 命令其实也是用户程序，不属于内核代码。这里用了一个循环，表示可以同时删除多个文件。`rm` 的使用个数如下：

```bash
rm file1 file2 ...
```

所以 `i=1` 表示从第二个参数开始删除。接下来我们查看 `unlink` 的实现，文件的系统调用实现一般在 `sysfile.c` 中，假设我们要删除 a，实现思路如下：

1. 找到 a 所在目录，用 `nameiparent` 实现
2. 若 a 是 `.` 或 `..` 报错返回（本身目录和父目录不可删）
3. 找到 a 的索引节点，用 `dirlookup` 实现
4. 若 a 为非空目录，则不可删，返回
5. 首先更新 a 所在的目录，清除记录 a 的目录项，并且所在目录的 `nlink` 减 1。
6. a 的 `nlink` 减 1。

每次修改索引节点，都会调用 `iupdate` 来更新对应磁盘上的索引节点。

这里还涉及到一个函数 `iunlockput()`，此函数会真正地释放内存索引节点和更新位图。

## 3. 恢复文件

我们可以发现 `rm` 将 `nlink` 减 1 后，立即调用了 `iunlockput` 函数，若索引节点的 `nlink` 为零，且 `ref` 也为零时，就会删除索引节点，更新位图，而索引节点代表文件，此时如果没有索引节点的索引功能，将找不到盘块的组织信息，也就恢复不了删除的文件。

启动 xv6，利用 `echo` 和重定向功能生成一个 `content` 文件。

```bash
echo hello world! > content
```

此时在 xv6 目录下多了个叫 content 的文件，查看其内容，如下：

```bash
content        2 19 13
$ cat content
hello world!
```

可知 content 的类型为 2（文件），索引节点为 19，文件大小为 13。

