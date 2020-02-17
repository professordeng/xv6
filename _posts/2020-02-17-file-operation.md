---
title: 2. 文件系统操作
date: 2019-09-03
---

本节使用自底向上的方式来讨论文件系统的操作：

1. 先讨论磁盘盘块的操作，此时没有文件系统的概念。
2. 然后是索引节点自身的访问，以及索引节点所管理的文件盘块的访问；这些操作都建立在盘块操作的基础之上。
3. 接着是目录文件内容的访问，它是在上一步通过索引节点访问普通文件的基础之上， 加上 `dirent` 目录项格式的解析。
4. 最后讨论进程进行的文件操作。

## 1. 盘块操作

因为磁盘是块设备，这意味着每次操作的最小单位就是盘块。所有磁盘文件系统相关的操作最终都落实为磁盘盘块的读写操作。 

由于磁盘的盘块读写操作比较慢（相对于处理器速度），因此使用内存中的块缓存来加块其访问速度，例如对一个盘块的多次写操作可以在内存中完成，直到换出到磁盘时才真正执行写盘操作。

因此讨论盘块的操作就必须讨论块缓存的问题。

### 1.1 块 IO 层

Linux 有比较完善的块 IO 层，不仅完成块缓存，包括对磁盘 IO 进行调度，以减少磁头移动距离，提高磁盘吞吐率。但是 `xv6` 的块 IO 层比较简单，仅有简单直接的块缓存，而与虚存管理、文件映射内存等高级特性无关。

`xv6fs` 的所有文件操作，都需要落到磁盘盘块的读写操作上，而 `xv6fs` 的磁盘盘块读写操作又结合了块缓存的概念。为了加速磁盘读写速度，`xv6fs` 与 Linux 文件系统一样，使用了 `block buffer` 块缓冲层。与 Linux 使用的页缓存（结合块缓存）并且具备文件映射页的方式不同，`xv6` 只有块缓存且没有所谓的文件映射页的概念。

由于数据局部性原理，访问过的文件数据很可能会被再次用到，因此把它们放到内存缓冲区中，后续需要的时候可快速访问到。如果读写的数据不在缓存里，才需要启动磁盘 `IDE` 硬件的读写操作，并将这些数据写入到块缓存中。当所有块缓存都缓存有数据时，再访问新的盘块时则需要将一些缓存中的回收利用，被回收的块缓存使用 `LRU` 算法来选择。

Linux 的文件系统允许直接访问磁盘而不经过页缓存（块缓存），但是 `xv6fs` 设计的比较简单，只提供经过块缓存的访问方式，不允许直接访问方式。

**块缓存**

`xv6` 在 [bio.c#L29](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L29) 定义了一个结构体变量 `bcache`，这个就是块缓存的管理数据结构，它管理了 [NBUF](https://github.com/professordeng/xv6-expansion/blob/master/param.h#L12) 个块缓存。 

单个块缓冲区使用 [buf.h](https://github.com/professordeng/xv6-expansion/blob/master/buf.h) 的 `buf` 结构体来记录。其中的盘块数据记录在 `buf.data[BSIZE]`（512 字节）成员中，所缓存的盘块号为 `blockno`，所在的设备由 `dev` 指 出，`prev` 和 `next` 用于和其他块缓存一起构建 `LRU` 链表，最后如果是脏的块缓存将通过 `qnext` 挂入到磁盘 `IO` 调度队列。 

块缓存在工作中会有不同状态，`B_BUSY` 表示正在读写中，`B_VALID` 表示里面缓存有盘块数据，`B_DIRTY` 表示里面缓存的盘块数据已经被改写过了。

**初始化**

在启动时由 `kernel` 的 `main()` 执行 `bcahce` 初始化函数 [binit()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L38)，`binit()` 它完成两件工作：

1. 创建 `bcache` 互斥锁。
2. 将这些块缓存 `buf` 构成一个 `LRU` 双向链表。表头为 `bcache->head`，注意 `bcache->head` 本身也是一个 `buf` 结构体。

块缓存被使用的时候用 `B_BUSY` 作为标记，用完之后要清除该标志，同时还把该块缓存移动到 LRU 链表的头部，从而实现 LRU 中的最近使用情况在链表中的排序。

### 1.2 块缓存操作

块缓存除了初始化之外，只有两个操作：

1. 根据设备和盘块号查找（获分配）块缓存的 `bget()`。
2. 释放块缓存的 `brelse()`。

**查找**

[bio.c#L58](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L58) 行定义了 `bget()`，它根据给定设备号和磁盘盘块号查找相应的块缓存，如果还未被缓存过，则分配一个块缓存。查找过程只是简单地扫描块缓存数组，比较块缓存所缓存的磁盘盘块号，从而发现是否有匹配的块缓存。如果和设备号和盘块号匹配的块缓存，但其 `b->flags` 的 `BUSY` 标志为 1，则必须等待其空闲才能使用。如果没有找到匹配的盘块，则找一个空闲的干净盘块，所谓干净指未被改写或已经写回到磁盘。

当块缓存数量很大时，这种线性搜索的方法是比较低效的，可以考虑使用 `hash` 链的方式来加快匹配搜索的过程。 

**释放**

[bio.c#L118](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L118) 定义了 `brelse ()`，它将释放指定的块缓存，将它插入到块缓存 `LRU` 头部。

`bio.c` 还定义了盘块的读写函数 `bread()` 和 `bwrite()`，两者都建立在 `IDE` 硬盘的盘块读写操作 `iderw()` 基础之上。

由于使用了块缓存，因此读写操作是通过块缓存完成，磁盘盘块所对应的块缓存是通过 `bget()` 函数查找到的（如果未找到则分配一个块缓存）。用过的块缓存通过 `brelse()` 释放，并插入到块缓存 `LRU` 链的开头，表示刚被使用过。

### 1.3 盘块的读写操作

盘块的读写操作 `bread()` 和 `bwrite()` 需要块缓存参与，因此定义在 `bio.c` 中；但是 `balloc()`、`bzero()` 和 `bfree()` 不涉及块缓存的操作，应此定义在 [fs.c](https://github.com/professordeng/xv6-expansion/blob/master/fs.c) 中。 

[bread()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L95)

当需要读入一个盘块数据的时候，`bread()` 首先调用 `bget()` 获得相应的块缓存。`bget()` 用于查找指定的某个盘块（设备号 + 盘块号）的块缓存，如果未找到则扫描 `LRU` 链表，找到一个不忙（`B_BUSY==0`）且不脏（`B_DIRTY==0`）的块缓存用于该盘块。因此当 `bget()` 返回时，一定带有一个块缓存，有可能是从 `LRU` 上查找到的带有盘块数据的（`B_VALID==0`），也可能是新分配的（`B_VALID==0`）。对于后者，因为没有有效数据，还需要调用设备驱动程序提供的 `iderw()`。

[bwite()](https://github.com/professordeng/xv6-expansion/blob/master/bio.c#L108)

`bwrite()` 用于将块缓存中的数据写回到对应的盘块中，具体执行也是通过 `IDE` 设备驱动程序提供的 `iderw()`。对比 `bread()` 和 `bwrite()` 的参数发现，`bread()` 的参数是盘块号，而 `bwrite()` 的参数是块缓存，无需先通过 `bget()` 获得块缓存（这是因为在 `writei()` 调用 [bwrite()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L479) 之前已经执行过对硬盘盘块的 `bread()` 了）。 

[readsb()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L30)

文件系统的超级块放在最开头的一个盘块中，记录了磁盘文件系统的整体上的布局信息。`readsb()` 从指定设备上读入超级块。超级块是编号为 1 的块，因此借助磁盘块的读入函数 `bread(dev, 1)` 读入到块缓存 `bp` 中，然后再通过 `memmove()` 从块缓存拷贝到内存超级块对象 `sb`（struct superblock）。 

[bzero()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L41)

`bzero()` 将指定盘块内容清零：首先调用 `bread()` 将盘块读入到块缓存 `bp` 中，然后将块缓存清 0，最后在用 `log_write()` 写回。

[balloc()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L55)

`balloc()` 分配一个空闲磁盘盘块，并将其内容清空。该函数扫描 `bmap` 位图，遍历（`0~sb.size`）所有的盘块号，由于一个 `bmap` 盘块可以记录 `BPB=BSIZE*8=512*8` 个位，因此盘块号每次递增 `BPB`。

对于每一个 `bmap` 盘块，逐个位进行检查是否为 0，找到后则将对应位置 1，表示已分配使用。然后用 `log_write()` 更新这个 `bmap` 盘块，最后用 `bzero()` 清空刚分配的盘块。

[bfree()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L80)

`bfree()` 用于释放一个磁盘盘块。其操作非常简单，就是将 `bmap` 位图中对应的位清 0 即可，其位图查找过程和前面 `balloc()` 相同。

理论上，`xv6` 删除文件后只是将数据盘块的位图清 0，而数据盘块本身并不进行清除， 因此其数据是未丢失的（因此数据是可以恢复的）。  

[fs.c](https://github.com/professordeng/xv6-expansion/blob/master/fs.c)

`fs.c` 的代码涉及三部分内容：

1. 盘块操作的代码
2. `inode` 操作的代码。
3. 目录操作的代码。

## 2. 索引节点的操作

在了解的磁盘布局和盘块操作之后，我们可以来讨论索引节点的问题。文件系统中最核心的对象就是索引节点，虽然 `inode` 没有文件名（文件名是目录的功能），但它代表并管理着文件在磁盘上的数据。目录操作也是建立在索引节点的操作之上的。 

用户进行文件读写操作时，使用的是文件逻辑偏移，而磁盘读写使用的是盘块号和盘块内偏移，只有索引节点才真正知道数据存储在哪个盘块上，即文件的物理结构。 

索引节点的操作分为两类： 

1. 对索引节点所管理的文件数据的读写。
2. 对索引节点自身的分配、删除和修改。

### 2.1 索引节点缓存

为了加快索引节点的读写操作，`xv6` 使用了索引节点的缓存，因此索引节点就呈现出 “磁盘索引节点” 和 “内存索引节点” 两个版本。

索引节点缓存 `icache` 定义在 [fs.c#L167](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L167)，它是磁盘索引节点在内存中的缓存， 可以加快索引节点的读写操作。磁盘索引节点位于磁盘中，这些索引节点缓存是磁盘索引节点在内存中的影像，并加上动态管理数据，例如引用计数 `ref` 和标志 `flags`。`icache.inode[NINODE]` 中最多可以缓存 `NINODE` 个磁盘索引节点的信息，其中的元素 `inode[]` 则是依据磁盘上的 `dinode` 结构体而填写。

**（内存）索引节点** 

`icache` 缓存的是多个 `inode` 内存索引节点，它是磁盘索引节点 `dinode` 的内存形态，定义于 [file.h#L12](https://github.com/professordeng/xv6-expansion/blob/master/file.h#L12)。

内存索引节点的前半部分是磁盘索引节点所没有的成员变量，后面的 `type`、`major`、`minor`、`nlink`、`size` 和 `addr[]` 则是完全从磁盘 [dinode](https://github.com/professordeng/xv6-expansion/blob/master/fs.h#L28) 中拷贝而来。 

索引节点缓存根据工作状态，分成已分配 `Allocation`、被引用 `Referencing`（`ref>0`）、有效 `Valid`（`flags` 的 `I_VALID` 置位）、锁定 `Locked`（`flags` 的 `I_BUSY` 置位）。 

**索引节点缓存初始化**

[iinit()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L172) 用于内存索引节点缓存的初始化。在对索引节点缓存的互斥锁完成初始化之后，读入超级块并打印出磁盘布局信息，这就是 `xv6` 启动时打印的内容之一，如下：

```bash
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58
```

很意外地发现 `iinit()` 是在 `forkret()` 中调用的，不过只被调用一次，这是由 `first` 变量保证的。这是因为 `iinit()` 和 `loginit()` 必须在进程环境中进行，因此不能放在 `kernel` 的 `main()` 初始化中执行。

## 2.2 索引节点上的 “文件读写” 操作

进程发出的文件读写操作，会转换到该文件索引节点所管理的盘块上的读写操作。我们先来学习对一个已经存在的文件，如果已经找到它的 `inode` 索引节点，如何读写该文件的内容数据。

[readi()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L450)

`readi()` 用于从 `inode` 对应的磁盘文件的偏移 `off` 处，读入 n 个字节到 `dst` 指向的数据缓冲区 中。如果是设备文件（`T_DEV`），则使用设备的读操作函数 `devsw[ip->major].read()` 完成读入操作。否则将执行磁盘文件的读入操作，略微有些复杂。

磁盘文件需要逐个盘块读入数据，但首先要知道文件偏移量对应的物理盘块号是哪个，这是通过 `bmap()` 完成的。 

确定盘块号之后，将会调用前面讨论过的 `bread()` 完成磁盘盘块的读入。由于 `bread()` 将数据读入到块缓存中，因此还需要用 `memmove()` 将数据拷贝到用户空间缓冲区。

[bmap()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L371)

由于进程发出的文件读写操作使用的是字节偏移（转换成文件内部的逻辑盘块号 `bn`）， 而磁盘读写 `bread()` 和 `bwrite()` 使用的是物理盘块号，因此需要 `bmap()` 将文件字节偏移对应的逻辑盘块号 `bn` 转换成物理盘块号。其转换过程需要借助索引节点的 `dinode.addr[]` 或 `inode.addr[]`，并且需要考虑直接盘块和间接盘块。

如果对应的数据盘块不存在，则 `bmap()` 会调用 `balloc()` 分配一个空闲盘块，然后再修改索引，使得 `ip->addrs[bn]` 指向新分配的盘块；如果该偏移落入间接索引区，则可能还需要分配间接索引盘块，然后才能分配 `bn` 所对应的数据盘块并建立索引关系。 

[writei()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L479)

`writei()` 需要逐个盘块写出数据，因为有块缓存的存在，因此会先调用 `bread()` 完成磁盘盘块的读入到块缓存，然后才是将数据拷贝到块缓存中，最后由 `log_write()` 向日志系统写出。 

如果是设备（`T_DEV`）则需要通过它自己的读函数 `devsw[ip->major].read` 其完成。 

[fs.c](https://github.com/professordeng/xv6-expansion/blob/master/fs.c)

`fs.c` 的代码涉及三部分内容：

1. 盘块操作的代码。
2. `inode` 操作的代码。
3. 目录操作的代码。 

本小节分析了盘块操作的代码以及 `inode` 操作中有关 “文件数据” 读写的代码，其余代码将陆续完成分析。 

### 2.3 对索引节点的操作

前面是对已有的文件、且已经定位其 `inode` 索引节点的读写操作。但是文件操作并不仅限于对已有文件的读写操作，还包括创建、删除等其他操作。

`xv6` 的索引节点操作同时涉及磁盘索引节点 `dinode` 及其缓存 `inode`（内存索引节点）。

[ialloc()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L191)

新创建一个文件时，就需要用 `ialloc()` 分配一个新的索引节点。`ialloc()` 用于在 `dev` 设备上分配一个类型为 `type` 的索引节点。该函数扫描整个磁盘索引节点区，遍历 `1~sb.ninodes` 的所有索引节点编号，逐个检查其类型 `type` 是否为 0，如果为 0 则设置其类型为 `type`，并用 `log_write()` 通知日志系统更新磁盘内容。

然后用 `iget()` 建立相应的索引节点缓存。也就是说 `ialloc()` 同时完成在磁盘上和内存缓存中的操作。由于是分配空闲的索引节点，因此无需从磁盘读入其 `dinode` 内容来填写 `inode` 缓存。 

[iget()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L239)

根据设备号 `dev` 和索引节点号 `inum` 在索引节点缓存中查找，返回所匹配的索引节点缓存， 或者分配一个空闲的索引节点缓存。 

`iget()` 工作过程如下：遍历所有 `icahce.inode[]` 缓存找到 `dev` 和 `inum` 所指定的 `inode` 则增 加其 `ref` 引用计数。若没有找到对应的 `inode` 缓存，但是有空闲的 `inode` 则记录在 `ip` 指针上， 然后填写该 `inode` 内容并返回。

如果 `iget()` 不仅没有找到对应的 `inode` 缓存，且发现没有空闲的缓存（`empty==0`），则应该回收 `inode` 缓存。

[iupdate()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L217) 

`iupdate()` 将 `inode` 缓存的内容更新到磁盘 `dinode` 上，最后写出到磁盘中。

[idup()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L275)

增加索引节点缓存的引用计数。 

[stati](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L438)

将索引节点的基本信息读入到 `stat` 结构体并返回。 

[itrunc()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L403)

将索引节点所管理文件数据的直接块和间接块都释放掉，每个磁盘盘块通过 `bfree()` 释放（数据盘块的位图清零）。

## 3. 目录操作

前面讨论的内容，要不就是盘块操作，要不就是已经获得文件的索引节点情况下的操作，它们都不需要文件名作为输入信息。现在来讨论如何根据文件名查找文件的索引节点，以及文件路径名构成的树形目录结构。

### 3.1 目录查找

 [dirlookup()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L523)、[namecmp](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L514)

根据文件名，在目录文件中查找目录项。字符串比较是通过 `namecmp()` 完成的（实际上就是简单封装了 `strncmp()`）。 

在文件目录的查找过程中，偏移量每次移动 `sizof(de)`。

[dirlink()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L551) 

创建一个链接文件，首先确认没有重名文件，其次找到一个空闲目录项，然后填写内容。

[skipelm](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L581)

用于逐级分解路径名。

## 4. 文件

`xv6` 中的文件，即 `struct file`，是对磁盘文件 `inode` 的一次动态抽象和封装。对 `file` 结构体 而言，文件内容时一个一维的字节数组，可以用一个简单的偏移量而访问到文件中的任意数据。向上与进程的文件描述符表建立联系，使得进程对自己所打开的文件使用一个简单的索引值来识别。向下与磁盘文件 `inode` 对象相关联，以便可以读写文件中的数据（定位数据所在的磁盘盘块号）。`file` 抽象的出的线性字节序列的文件和磁盘上乱序存储的盘块的联系，是通过 `inode` 的索引地址进行映射的。

### 4.1 文件对象与文件描述符

**“文件 file” 与 “索引节点 inode”**

文件 `file` 特指磁盘文件的一次动态打开，是动态概念。而 `inode` 代表的磁盘文件则是静态概念。每一此打开的文件都是用 `file` 结构体（见 [file.h](https://github.com/professordeng/xv6-expansion/blob/master/file.h)）来描述， 主要记录该文件的类型、引用数、是否可读、是否可写，如果是管道则由 `pipe` 成员指向管道数据，`ip` 指向内存中的 `inode`（是磁盘 `inode` 在内存的动态存在）。`off` 用于记录当前读写位置，通常称作文件的游标。

磁盘文件 `dinode` 作为静态内容，被打开之后在内存中创建一个 `inode` 对象（动态），其主要内容来源于磁盘的的 `dinode`。例如 `inode` 需要有 `inum` 记录 `inode` 号，而磁盘上 `dinode` 的则每个 `dinode` 都唯一对应一个 `inode` 号。[file.h#L12](https://github.com/professordeng/xv6-expansion/blob/master/file.h#L12) 的 `inode` 结构体成员，和磁盘上的 `dinode` 完全一致，直接从磁盘读入即可。内存中的 `inode` 对象由 `icache` 缓冲区管理，内含 `inode[NINODE]` 数组，其中 `NINODE=50`。

也就是说 `xv6fs` 和 Linux 类似，有内存 `inode` 和相对应的磁盘 `dinode`。Linux 则是内存 `inode` 是 VFS 的抽象 `inode`，而磁盘上的则可以是各种具体不同的如 `ext2` 的 `inode`、`xfs` 的 `inode` 等。 

**“已打开文件列表” 和 "文件描述符表"**

各进程内部使用一个 “文件描述符表” 来管理自己所打开的文件，而 “已打开文件列表” `ftable` 是全系统已打开文件的列表。前者是一个简单的引用，指向一打开文件的的 `file` 结构体， 后者则是 `file` 结构体自身。多个文件描述符可能指向同一个已打开文件，例如终端设备会被多个程序所使用。 

[file.h](https://github.com/professordeng/xv6-expansion/blob/master/file.h)

在 `file.h` 的一开头就定义了 `file` 结构体，用于一个动态概念的已打开文件，前面刚刚分析过。

虽然设备都抽象为设备文件，但是各自的读写操作细节是不同的，因此使用了 `devsw[]` 数组来记录给出的读写操作函数，用设备号来索引。从这里可以看出 `xv6` 的设备操作非常简陋， 只有读写操作，而没有 Linux 设备中的 `ioctl()` 等操作。

### 4.2 file 对象的操作

[fileinit()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L19) 用于对已打开文件列表 `ftable[]` 进行初始化，即初始化 `ftable.lock` 自旋锁。 

[filealloc()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L25) 分配一个空闲的 `file` 对象。它扫描 `ftable.file[]`，如果发现未使用（即 `ref==0`），则将其标记为已分配使用（`ref=1`），并返回该 `file` 指针。

当用户进程对某个文件描述符进行复制时，将引起对应的 `file` 对象引用次数增 1，因此 [filedup()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L43) 就是将制定的 `file` 对象的 `file->ref ++`。

[fileclose()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L55) 要考虑多个进程打开文件的情况，因此如果不是最后一个使用该文件的进程，只需要简单地将 `file->ref` 减一即可。否则，将完成真正的关闭操作，对于 `PIPE` 管道将使用 `pipeclose()`，对其他文件则通过 `iput()` 释放相应的索引节点。

[filestat()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L82) 用于读取文件的元数据（`meta data`），它通过 `stati()` 将元数据管理信息填写到 `stat` 结构体中。

 **文件读写操作**

[fileread()](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L95) 是通用的文件读操作函数，它内部再根据文件类型（`FD_PIPE`、`FD_INODE`）区分执行 `piperead()` 和 `readi()` 两种操作。对于磁盘文件，需要定位数据偏移所对应的磁盘盘块所在的位置，这个就必须通过 `readi()` 来完成。 

在 [stat.h](https://github.com/professordeng/xv6-expansion/blob/master/stat.h) 中定义了三种文件类型 `T_DIR` 目录、`T_FILE` 普通文件和 `T_DEV` 设备文件。其次是 `stat` 结构体。

[file.c](https://github.com/professordeng/xv6-expansion/blob/master/file.c) 涉及文件系统中关于 `file` 层面的操作，上接系统调用代码的接口层面的代码，向下衔接 `inode` 操作层面的操作代码。

系统中所有已经打开的文件记录在 [ftable](https://github.com/professordeng/xv6-expansion/blob/master/file.c#L14) 中，由于 `ftable.file[NFILE]` 中 `NFILE=100`，因此 `xv6` 系统最多允许打开 100 个文件。

## 5. 系统调用

文件系统的用户态接口是在 [usys.S](https://github.com/professordeng/xv6-expansion/blob/master/usys.S) 中定义的，大部分在 [sysfile.c](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c) 中实现，也就是说用户的 C 程序中使用 `open()`、`close()`、`dup()`、`fstat()`、`read()`、`write()`、`mkdir()`、`mknod()`、`chdir()`、`link()`、`unlink()` 等形式的宏， 就可以进行相应的系统调用。

经过系统调用的公共入口之后，将分别转到 `sys_open()`、`sys_close()`、`sys_dup()`、`sys_fstat()`、`sys_read()`、`sys_write()`、`sys_mkdir()`、`sys_mknod()`、`sys_chdir()`、`sys_link()`、`sys_unlink()` 等具体系统调用服务函数中。

### 5.1 文件打开与关闭

1. 文件描述符

   `fdalloc()` 将扫描本进程的文件描述符数组 `proc->ofile[]`，如果其值为 0 则表示空闲可用，将它指向文件对象即可。

2. 文件对象

   `file.c` 文件中的 `filealloc()` 用于分配文件对象，它将扫描系统的 `ftable.file[]`（共有 `NFILE` 个元素）数组，找到一个空闲的 `file` 对象然后将其引用计数 `ref` 增 1，表示被使用。 

**具体操作**

[sys_open()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L286) 用于打开文件的通用操作，并不区分磁盘文件、设备文件或管道文件。这些文件的区别在文件读写操作的时候。

如果打开模式带有 `O_CREATE` 标志，则使用 `create()` 打开文件。

否则，先通过 `namei()` 查找相应的索引节点，然后由 `filealloc()` 分配文件对象，最后 `fdalloc()` 为该文件对象创建描述符。

### 5.2 文件读写操作

[sys_read()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L69) 首先检查参数是否正确，然后调用 `fileread()` 完成读入操作。 

[sys_write()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L81) 首先检查参数是否正确，然后调用 `filewrite()` 完成读入操作。

### 5.3 目录操作

[sys_mkdir()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L336) 在文件系统的目录树中创建一个目录节点，它是通过调用 `create()` 创建相应的索引节点的，需要指定类型为目录 `T_DIR`。

[sys_mknod()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L352) 在文件系统的目录树中创建一个设备节点，共有三个参数：第一个参数是该节点所在的路径名，第二个参数是设备的主设备号 `major`，第三个参数是设备的次设备号 `minor`。具体操作的完成是通过 `create()` 完成的。

[sys_chdir()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L372) 将改变进程的当前目录，即 `proc->cwd`。它用 `argstr()` 从调用参数中获得新的路 径，然后借助 `namei()` 找到对应的索引节点。如果索引节点的类型是目录 `T_DIR`，则将本进程的工作目录 `proc->cwd` 设置为该索引节点。

### 5.4 其他操作

[sys_pipe()](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c#L423) 对应的用户态函数原型是 `user.h` 中的 `int pipe(int *)`，其中 `int *` 指向连续的两个整数，用做输入和输出两个文件描述符。`sys_pipe()` 内部主要就是通过 `pipealloc(&rf,&wf)` 分配两个 `file` 结构体，然后为它们分配文件描述符 `fd0` 和 `fd1`，最后将这两个描述符返回给用户进程。 `pipealloc()` 函数定义于 [pipe.c#L22](https://github.com/professordeng/xv6-expansion/blob/master/pipe.c#L22)。

### 5.5 代码总结

[sysfile.c](https://github.com/professordeng/xv6-expansion/blob/master/sysfile.c) 实现的文件系统接口，因此并不区分磁盘文件、管道或设备。另外文件的写操作部分，会和日志 `log` 代码有互动。文件系统层的抽象操作代码，向下将经过 `file` 层面代码，进入到 `inode` 层面的代码，然后根据文件类型不同而转向不同的处理函数：可能是管道、磁盘文件或设备的操作代码。

作为文件系统的最顶层，向上与应用进程交互，这需要通过文件描述符作为中介对象。每个进程维护着自己所打开的文件描述符列表，文件描述符只是对所打开文件的一个引用。`fdalloc()` 在打开文件时用来获取本进程的一个空闲文件描述符，其过程很简单，就是扫描本进程的文件描述符数组 `proc->ofile[]`，找到未使用的一个，并将其指向文件的 `file` 结构体。

 