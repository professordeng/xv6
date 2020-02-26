---
title: 2. 磁盘盘块操作
---

我们使用自底向上的方式来讨论文件系统的操作和代码实现：

1. 先讨论磁盘盘块的操作，此时没有文件系统的概念。
2. 然后是索引节点自身的访问，以及索引节点所管理的文件盘块的访问；这些操作都建立在盘块操作的基础之上。
3. 接着是目录文件内容的访问，它是在上一步通过索引节点访问普通文件的基础之上， 加上 [dirent](-drive file=rawdisk.img,index=2,media=disk,format=raw) 目录项格式的解析。
4. 最后讨论进程进行的文件操作。

---

本节只讨论盘块操作层，只涉及盘块的读写，没有文件系统概念。

因为磁盘是块设备，这意味着每次操作的最小单位就是盘块。所有磁盘文件系统相关的操作最终都落实为磁盘盘块的读写操作。 

由于磁盘的盘块读写操作比较慢（相对于处理器速度），因此使用内存中的块缓存来加块其访问速度，例如对一个盘块的多次写操作可以在内存中完成，直到换出到磁盘时才真正执行写盘操作。

因此讨论盘块的操作就必须讨论块缓存的问题。

## 1. 块 IO 层

Linux 有比较完善的块 IO 层，不仅完成块缓存，包括对磁盘 IO 进行调度，以减少磁头移动距离，提高磁盘吞吐率。但是 XV6 的块 IO 层比较简单，仅有简单直接的块缓存，而与虚存管理、文件映射内存等高级特性无关。

XV6 文件系统的所有文件操作，都需要落到磁盘盘块的读写操作上，而 XV6 文件系统的磁盘盘块读写操作又结合了块缓存的概念。为了加速磁盘读写速度，XV6 文件系统 与 Linux 文件系统一样，使用了 [block buffer](https://github.com/professordeng/xv6-expansion/blob/master/buf.h#L1) 块缓冲层。与 Linux 使用的页缓存（结合块缓存）并且具备文件映射页的方式不同，XV6 只有块缓存且没有所谓的文件映射页的概念。

由于数据局部性原理，访问过的文件数据很可能会被再次用到，因此把它们放到内存缓冲区中，后续需要的时候可快速访问到。如果读写的数据不在缓存里，才需要启动磁盘 IDE 硬件的读写操作，并将这些数据写入到块缓存中。当所有块缓存都缓存有数据时，再访问新的盘块时则需要将一些缓存中的回收利用，被回收的块缓存使用 LRU 算法来选择。

Linux 的文件系统允许直接访问磁盘而不经过页缓存（块缓存），但是 XV6 文件系统设计的比较简单，只提供经过块缓存的访问方式，不允许直接访问方式。

### 1.1 块缓存

XV6 定义了一个 [bcache](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L29) 结构体变量，这个就是块缓存的管理数据结构，它管理了 [NBUF](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L12) 个块缓存。 

单个块缓冲区使用 [buf](https://github.com/professordeng/xv6-expansion/blob/master/buf.h) 结构体来记录。其中的盘块数据记录在 `data[BSIZE]`（512 字节）成员中，所缓存的盘块号为 `blockno`，所在的设备由 `dev` 指 出，`prev` 和 `next` 用于和其他块缓存一起构建 `LRU` 链表，最后如果是脏的块缓存将通过 `qnext` 挂入到磁盘 IO 调度队列。 

块缓存在工作中会有不同状态，`B_BUSY` 表示正在读写中，`B_VALID` 表示里面缓存有盘块数据，`B_DIRTY` 表示里面缓存的盘块数据已经被改写过了。

### 1.2 初始化

在启动时由 kernel 的 `main()` 执行 `bcahce` 初始化函数 [binit()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L38)，`binit()` 它完成两件工作：

1. 创建 `bcache` 互斥锁。
2. 将这些块缓存 `buf` 构成一个 `LRU` 双向链表。表头为 `bcache->head`，注意 `bcache->head` 本身也是一个 `buf` 结构体。

块缓存被使用的时候用 `B_BUSY` 作为标记，用完之后要清除该标志，同时还把该块缓存移动到 LRU 链表的头部，从而实现 LRU 中的最近使用情况在链表中的排序。

## 2. 块缓存操作

块缓存除了初始化之外，只有两个操作：

1. 根据设备和盘块号查找（或分配）块缓存的 [bget()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L58)。
2. 释放块缓存的 [brelse()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L118)。

### 2.1 查找

`bget()` 根据给定设备号和磁盘盘块号查找相应的块缓存，如果还未被缓存过，则分配一个块缓存。查找过程只是简单地扫描块缓存数组，比较块缓存所缓存的磁盘盘块号，从而发现是否有匹配的块缓存。如果和设备号和盘块号匹配的块缓存，但其 `b->flags` 的 `DIRTY` 标志为 1，则必须等待其空闲才能使用。如果没有找到匹配的盘块，则找一个空闲的干净盘块，所谓干净指未被改写或已经写回到磁盘。

当块缓存数量很大时，这种线性搜索的方法是比较低效的，可以考虑使用 `hash` 链的方式来加快匹配搜索的过程。 

### 2.2 释放

`brelse ()` 将释放指定的块缓存并插入到块缓存 LRU 头部。

`bio.c` 还定义了盘块的读写函数 [bread()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L95) 和 [bwrite()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L108)，两者都建立在 IDE 硬盘的盘块读写操作 [iderw()](https://github.com/professordeng/xv6-expansion/blob/master/ide.c#L133) 基础之上。

由于使用了块缓存，因此读写操作是通过块缓存完成，磁盘盘块所对应的块缓存是通过 `bget()` 函数查找到的（如果未找到则分配一个块缓存）。用过的块缓存通过 `brelse()` 释放，并插入到块缓存 LRU 链的开头，表示刚被使用过。

## 3. 盘块的读写操作

盘块的读写操作 `bread()` 和 `bwrite()` 需要块缓存参与，因此定义在 `bio.c` 中；但是 `balloc()`、`bzero()` 和 `bfree()` 不涉及块缓存的操作，应此定义在 [fs.c](https://github.com/professordeng/xv6-expansion/blob/master/fs.c) 中。 

[bread()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L95)

当需要读入一个盘块数据的时候，`bread()` 首先调用 `bget()` 获得相应的块缓存。`bget()` 用于查找指定的某个盘块（设备号 + 盘块号）的块缓存，如果未找到则扫描 LRU 链表，找到一个不忙（`B_BUSY==0`）且不脏（`B_DIRTY==0`）的块缓存用于该盘块。因此当 `bget()` 返回时，一定带有一个块缓存，有可能是从 LRU 上查找到的带有盘块数据的，也可能是新分配的。对于后者，因为没有有效数据，还需要调用设备驱动程序提供的 [iderw()](https://github.com/professordeng/xv6-expansion/blob/master/ide.c#L133)。

[bwite()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L108)

`bwrite()` 用于将块缓存中的数据写回到对应的盘块中，具体执行也是通过 IDE 设备驱动程序提供的 `iderw()`。对比 `bread()` 和 `bwrite()` 的参数发现，`bread()` 的参数是盘块号，而 `bwrite()` 的参数是块缓存，无需先通过 `bget()` 获得块缓存（这是因为在 [writei()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L479) 调用 `bwrite()`之前已经执行过对硬盘盘块的 `bread()` 了）。 

[readsb()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L30)

文件系统的超级块放在最开头的一个盘块中，记录了磁盘文件系统的整体上的布局信息。`readsb()` 从指定设备上读入超级块。超级块是编号为 1 的块，因此借助磁盘块的读入函数 `bread(dev, 1)` 读入到块缓存 `bp` 中，然后再通过 `memmove()` 从块缓存拷贝到内存超级块对象 `sb`（struct superblock）。 

[bzero()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L41)

`bzero()` 将指定盘块内容清零：首先调用 `bread()` 将盘块读入到块缓存 `bp` 中，然后将块缓存清 0，最后在用 `log_write()` 写回。

[balloc()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L55)

`balloc()` 分配一个空闲磁盘盘块，并将其内容清空。该函数扫描 `bmap` 位图，遍历（`0~sb.size`）所有的盘块号，由于一个 `bmap` 盘块可以记录 `BPB=BSIZE*8=512*8` 个位，因此盘块号每次递增 `BPB`。

对于每一个 `bmap` 盘块，逐个位进行检查是否为 0，找到后则将对应位置 1，表示已分配使用。然后用 `log_write()` 更新这个 `bmap` 盘块，最后用 `bzero()` 清空刚分配的盘块。

[bfree()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L80)

`bfree()` 用于释放一个磁盘盘块。其操作非常简单，就是将 `bmap` 位图中对应的位清 0 即可，其位图查找过程和前面 `balloc()` 相同。

理论上，XV6 删除文件后只是将数据盘块的位图清 0，而数据盘块本身并不进行清除， 因此其数据是未丢失的（因此数据是可以恢复的）。  

[fs.c](https://github.com/professordeng/xv6-expansion/blob/master/fs.c)

`fs.c` 的代码涉及三部分内容：

1. 盘块操作的代码
2. `inode` 操作的代码。
3. 目录操作的代码。

 