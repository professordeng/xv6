---
title: xv6 虚存实验
---

## 文件系统实验

`xv6` 的文件系统中对磁盘的划分类似 `ext2` 底层划分，从底层到调用，`xv6` 的文件系统分 6 层实现。本文主要结合文件系统和内存管理知识给 `xv6` 添加虚拟内存功能。

## 磁盘裸设备的读写

1. 准备磁盘镜像文件 `rawdisk.img` ，大小为 `RAWSPACE_SIZE=32MB` 。

2. 修改 `Makefile` 的 `qemu` 仿真命令选项 `QEMUOPTS` ，加入下面语句

   ```bash
   -drive file=rawdisk.img,index=2,media=disk,format=raw
   ```

3. 系统启动时对 `rawdisk` 盘的初始化

   这里可以自己初始化布局，也可以参照 `ext2` 的磁盘布局。

4. 提供系统调用对 `rawdisk` 的读写

   ```
   // syscall.h
   
   ```

   

5. 编写应用程序检验读写结果：检查 `rawdisk.img` 内容是否符合所写结果

   这里可以利用 `dd` 指令直接读取。

## 虚拟内存

我们仅对进程空间中堆栈后面（高地址）的 `sbrk()` 分配的空间，允许其换出，堆栈前面的内容不换出（`text` 和 `data` 和 `bss`）。所以在 `proc` 结构体增加一个 `int swap_start` 成员，用于记录可交换空间的起始地址，具体是在 `exec.c` 的 `exec()` 建立堆栈时将当时的 `sz` 记录下来，下一个页边界就是将来 `sbrk()` 分配的空间，也就是可交换空间的起始地址。

### 1. 建立交换空间

1. 准备交换空间 `swapdisk.img` ，大小为 `SWAPSPACE_SIZE=32MB` （最大不能超过 `1M` 个盘块，一个盘块为 `4K` 字节（4个字节组成一个 32 位，因此有 `1K` 个 32 位），因为我们用页表项的高 20 位来记录盘块号）。

   ```bash
   dd if=/dev/zero of=swapdisk.img bs=4k count=8192
   ll swapdisk.img # 查看是否创建成功
   ```

   

2. 修改 `Makefile` 的 `qemu` 仿真命令选项 `QEMUOPTS` ，加入下面语句

   ```bash
   -drive file=swapdisk.img,index=2,media=disk,format=raw
   ```

   然后执行 `make qemu-nox` 查看是否可以运行成功，如果可以说明磁盘挂载成功，磁盘标志是 2。

3. 实现 `swapdiesk` 的正常读写操作

   首先实现 `pd` 系统调用来输出一些关于磁盘文件的信息。

   ```c
   // syscall.h
   #define SYS_pd 22
   
   // syscall.c
   extern int sys_pd(void);
   [SYS_pd]      sys_pd,
   
   // usys.S
   SYSCALL(pd)
   
   // sysproc.c
   int 
   sys_pd(void){
     	int dev;
      	if(argint(0, &dev) < 0)
   	 	return -1;
       pd();
       return 1;
   }
   
   // defs.h
   void            pd(void);
   
   // proc.c
   void pd(){
      	cprintf("swapfile free blocks: %d\n", swapfile.free_blocks);
       cprintf("swapfile total blocks: %d\n", swapfile.total_blocks);
   }
   
   // user.h
   int pd(int);
   
   // Makefile
   _pd\
   
   // pd.c
   #include "types.h"
   #include "stat.h"
   #include "user.h"
   
   int
   main(int argc, char *argv[])
   {
     if(argc <= 1){
       printf(1, "usage: pd [dev]");
       exit();
     }
     pd(atoi(argv[1]));
     exit();
   }
   
   ```

   

4. 对 `swapdisk` 进行格式化，形成以下布局

   **位图区** ：位区图大小需要根据交换空间总量，自动计算完成。总之要有足够的 `bit` 能记录所有的 `swapdisk` 中的数据盘块是否被占用。一个盘块是 `4K` 个字节，可以记录 `32K` 个盘块。而 `32M` 只有 `8192` 个盘块，位区图只需要一个盘块记录。

   这里为了更简单的实现，我们采用了一个字节（而不是一个位），所以位图区需要两个盘块。而且位图会存储在内存中，所以我们这里可以不需要位图区。

   **数据区** ：除了位图区，其他都是数据区，用来存储从内存中交换出来的数据。盘块的大小和物理页帧一样，都是 4096 个字节。

   交换分区使用以下结构体表示

   ```c
   struct swap_disk{
     int total_blocks; // 总的数据盘块数量
     int free_blocks;  // 空闲盘块
     char bitmap[BITMAP_BYTES]; // BITMAP_BYTES 根据 SWAPSPACE_SIZE 算出
     spinlock lock;
   } 
   ```

   `main.c` 的 `main()` 初始化时，`userinit()` 之前执行 `initswap()` 进行初始化锁、清除交换盘位图。

   搜索空闲位图使用 `int alloc_swap_block()` ，它扫描交换盘位图，找到第一个空闲盘块号，将位图对应位置 1，返回盘块号。反之，根据盘块号清除指定位图用 `int clear_swap_block(int block_num)` 完成。

   ```c
   // main.c
   initswap();  // 初始化锁, 清除交换盘位图
   
   // defs.h
   void initswap(void);
   int alloc_swap_block();
   void clear_swap_block(int);
   
   // spinlock.h
   
   #define BITMAP_BYTES 8192
   struct swap_disk{
      	int total_blocks;          // 总的数据盘块数量
      	int free_blocks;           // 空闲盘块
       char bitmap[BITMAP_BYTES]; // BITMAP_BYTES 根据 SWAPSPACE_SIZE 算出，这里是 8192
     	struct spinlock lock;
   };
   extern struct swap_disk swapfile;
   
   // proc.c
   struct swap_disk swapfile;
   void initswap() {
    	int i;
     	swapfile.free_blocks = BITMAP_BYTES;
      	swapfile.total_blocks = BITMAP_BYTES;
      	initlock(&swapfile.lock, "交换空间");
     	for(i = 0; i < BITMAP_BYTES; i++)
     		swapfile.bitmap[i] = 0;
   }
   
   int alloc_swap_block() {
     	int i;
       acquire(&swapfile.lock);
       for(i = 0; i < BITMAP_BYTES; i++)
   	  	if(swapfile.bitmap[i] == 0)
           	break;
       swapfile.bitmap[i] = 1;
       release(&swapfile.lock);
       return i;
   }
   
   void clear_swap_block(int block_num) {
      	acquire(&swapfile.lock);
       swapfile.bitmap[block_num] = 0;
       release(&swapfile.lock);
   }
   
   ```

   

### 2. 换出机制

在每次 `kalloc()` 分配物理页帧时，检查空闲物理页帧数量是否少于 `FREELIST_LOW=32` 如果是则释放一个进程的空间的 `sbrk()` 分配的空间的一个页。

为了知道剩余页帧数量，需要修改 `kmem` 结构体，加上一个 `free_frames` 计数值，记录空闲页帧的个数，每次 `kalloc()` 和 `kfree()` 后做相应的修改。

``` c
// defs.h
void            pk(void);

// kalloc.c
#define FREELIST_LOW 32
int free_frames;  // kmem结构体中添加。
void pk(){
   	cprintf("kmem free_frames: %d\n", kmem.free_frames);
}

kmem.free_frames = 0;        // kinit1 中添加

kmem.free_frames ++;        // kfree 中添加

kmem.free_frames --;        // kalloc 中添加

// proc.c
pk();      // pd 函数中添加       
```

换出某个进程的一个页帧的方法是

1. 在交换区起始地址 `proc->swap_start` 到 `proc->sz` 区间，找到有映射的页帧（walk），解除页表映射；为了知道某个虚页是否有映射，可以用 `walkpgdir()` 查找对应的 `PTE` 项，然后用 `(*pte)&PTE_P != 0` 检验是否有映射。如果一个进程没有可换出的页帧，查找下一个进程。

   ```c
   // proc.h
   int swap_start;
   
   // exec.c
   proc->swap_start = sz;
   
   // defs.h
   void pd(int);  //修改参数
   
   // proc.c
   sti();  // 可中断
   struct proc *p; 
   acquire(&ptable.lock);
   for(p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
   	if(p->pid != pid) continue;
   	cprintf("proc size: %d\n", p->sz);
   	cprintf("proc swap_start: %d\n", p->swap_start);
   }
   release(&ptable.lock);
   ```

   

2. 将页帧内容写入到 `swapdisk` 的某一个空闲盘块中。用 `int alloc_swap_block()` 查找一个空闲交换盘块（同时修改位图）并返回盘块号，`-1` 表示没有盘块了。获得盘块号后可以用 `bwrite()` 将物理页帧的内容写入磁盘。

3. 记录页帧的去向，以便将来换入内存。将换出的盘块号记录在页表项 `PTE` 中。此时 `PTE` 的 `present` 位置零，同时释放该物理页帧。高 20 位记录该页帧在 `swapdisk` 上的盘块号。

   |                           | 高 20 位                   | Page flags |
   | ------------------------- | -------------------------- | ---------- |
   | 有映射的页表项 `PTE` 内容 | `Page physical address`    | `PTE_P=1`  |
   | 换出的页表项 `PTE` 内容   | `Block number in swapdisk` | `PTE_P=0`  |

   注意：交换盘的盘块是和一个页帧等大小的 `4K`，而 `bread()/bwrite()` 的盘块是 512 字节，需要做一些适配性的代码工作。

### 3. 换入机制（缺页异常）

首先需要在中断机制中增加处理缺页异常。

其次在缺页异常处理的代码中，需要区分两种情况：

1. 如果缺页地址大于 `sz` 则表示非法地址，终止程序。
2. 如果地址在 `proc->swap_start` 到 `proc->sz` 以内，则是合法的可交换地址，进行处理。
   - 检查页表项 `PTE` 是否无效，如果 `PTE` 有映射，说明是其他缺页异常（例如访问权限），不处理直接终止程序。
   - 如果 `PTE` 无映射，且盘块号不为 0 ，说明是换出页，重新装入即可（借鉴 `allocuvm()` 和 `loaduvm()` 函数代码 ）
3. 转入后，将 `swapdisk` 中的对应位图清 0，表示该盘块可以接收其他换出内容。

### 4. 功能验证

编写应用程序，一个父进程创建两个子进程，子进程用 `sbrk()` 分配总量超过出 `240MB` 的物理内存（`PHYSTOP`）。各自往内存空间中写入特定数据，并计算个字节的 `hash` 值，以检验结
果正确性。对比的是在没有分页机制的系统上分别运行两种数据，获知正确的 `hash` 值作为对
比。

检查 `swapdisk.img` 内容，查找是否出现我们写入的特定数据（例如重复的 `SZU data pattern.` ）。