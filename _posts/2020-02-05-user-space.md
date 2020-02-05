---
title: 4. 进程用户空间
date: 2019-05-07
---

我们将进程（虚存）空间的初始化进行了分割（初始化和运行时动态的内存管理）。进程空间初始化又区分为第一个进程的用户空间创建过程和其他进程的用户空间创建过程。进程通过内存分配和回收的操作则称为运行时的用户空间操作。但是实际初始化过程和运行操作都是依赖 于 `allocuvm()`、`deallocuvm()` 两个函数，因此我们在这里一起讨论。 

## 1. 用户空间镜像

用户空间和内核空间在地址编码上是连续的，但是两个地址区间上的访问模式不同：内核空间的保护级为 0（最高），而用户空间为 3（最低）。也就是说用户态代码是不能通过地址指针指向内核空间而访问内核数据，同样也不能通过跳转而转向去执行内核代码。

### 1.1 用户空间布局

一个进程的页表所能看到的整个虚存空间如下图所示，用户空间是从可执行文件而创建的。 

![virtual address](/xv6-book/img/va.png)

### 1.2 init 用户进程空间

在第一个进程 `init` 创建以及 `exec()` 创建新进程镜像时，需要创建进程用户空间，准备装载代码和数据的页帧以及建立相应的页表映射。进程的页表在使用前需要初始化，其中必须覆盖自己的代码和数据区域，以及内核代码的映射（即和 `scheduler` 看到一样的内核空间）。进程用户代码和数据使用虚拟地址空间的低地址部分，高地址部分留给内核。

在初始化操作 `main()->userinit()` 创建第一个进程 `init` 的时候，其中的用户空间的建立是通 过 `main()->userinit()->inituvm()` 完成的。`inituvm()` 函数定义于 [vm.c#L180](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L180)，它分配一 个空闲页帧并获得相应的虚地址 `mem`，然后将该物理页帧映射到用户空间 0 地址的位置。也就是说此时该物理页帧由两个虚地址页像映射，然后再将 `initcode` 的代码拷贝到这个页帧内（通过 `mem` 地址）。于是 在用户空间地址 0 所对应的位置有了 `initcode.S` 的完整代码。

也就是说，`init` 进程的用户空间并非是从磁盘中获得的。 `initcode.S` 参与内核编译，因此它的代码在内核 `kernel` 的 ELF 文件中。我们通过 `objdump` 查看 ELF 符号可以看到如下结果，可以将 `_binary_initcode_start` 符号对应于地址 `0x8010a460`，位于内核空间中。   

```bash
➜  xv6-expansion git:(dev) objdump -t kernel | grep initcode
8010a48c g       .data	00000000 _binary_initcode_end
8010a460 g       .data	00000000 _binary_initcode_start
0000002c g       *ABS*	00000000 _binary_initcode_size
```

当 `initcode.S` 对应的进程执行后，立刻通过 `exec()` 系统调用转而执行磁盘上的 `/init` 程序， 从而称为真正的 `init` 进程（由 `init.c` 产生，之后的 `init.c` 分析）。 

## 2. 分配与回收

`xv6` 进程空间是一个连续区域，只能进行扩展和搜索两种操作。

### 2.1 allocuvm() & deallocuvm()

`allocuvm()`、`deallocuvm()` 负责完成用户进程的内存空间分配（扩展）和撤销（收缩）。[vm.c#L219](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L219) 的 `allocuvm()` 在设置页表的同时还需要分配相应的物理页帧供用户进程使用。 `allocuvm()` 中的参数 `oldsz`、`newsz`：这两个参数指出原来的 `0~oldsz` 到现在的 `0~newsz`。因此需要分配虚拟地址 `oldsz` 到 `newsz` 的以页为单位的内存。`allocuvm()` 从 `oldsz` 到 `newsz` 范围内逐个页进行处理，通过 `kalloc()` 分配一个空闲页帧然后再通过 `mappages()` 建立页表映射。 从这里可以看出，分配用户空间（虚存）的同时也分配了物理页帧。并不会出现所谓的 “缺页” 现象，因为 `xv6` 并没有真正意义上虚存页的换出和换入操作。 

`deallocuvm()`（见 [vm.c#L251](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L251)）则相反，它将 `newsz` 到 `oldsz` 对应的虚拟地址空间内存置为空闲。具体操作是用 `walkpgdir()` 找到相应的 `PTE`，然后将其 `PTE_P` 位清零表示没有映射，最后通过 `kfree()` 释放页帧。

 ### 2.2 其他进程用户空间 

`fork` 所产生的进程是拷贝父进程的所有资源的，因此通过拷贝父进程的页表内容。而 `exec()` 创建的进程则会·根据 ELF 可执行文件的装入要求，逐个段装入：使用 `vm.c` 文件中提供的 `loaduvm()`（[vm.c#L195](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L195)）将文件系统上的 `i` 节点内容（可执行文件）读取载入到相应的地址上，前面已经在相应的空间上已经映射了物理页帧。也就是说 `exec()` 前面的操作已经通过 `allocuvm()` 接口为用户进程分配内存和设置页表，然后才能调用 `loaduvm()` 接口将文件系统上的程序载入到内存。`loaduvm()` 用于支撑 `exec()` 系统调用，为指定的可执行文件创建用户进程的正式运行做准备。 

为了支持 `fork` 系统调用，在 `vm.c` 中提供了 `copyuvm()`（[vm.c#L313](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L313)）用于从父进程复制出一个新的页表并分配新的内存，新的内存布局（包括内核和用户空间）和父进程的完全一样。 

### 2.3 sbrk 系统调用

`xv6` 提供了 `sbrk` 系统调用，用于调整进程空间的大小。[sysproc.c#L45](https://github.com/professordeng/xv6-expansion/blob/master/sysproc.c#L45) 给出了 `sys_sbrk()` 系统调用的实现，它接收参数 n 并通过 `growproc(n)` 调整用户空间的大小。其中 n 为正数表示扩展，n 为负数表示收缩空间。`growporc()` 定义于 [proc.c#L156](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L156)，如果 n 为正数则调用 `allocuvm()` 扩大进程空间，否则调用 `deallocuvm()` 减小内存空间。最后还需要用 `switchuvm()` 切换到新的进程镜像中，让内存修改变化生效。 

### 2.4 撤销进程空间

当进程销毁需要回收内存时，可以调用 `freevm()` 清除用户进程相关的内存环境，`freevm()` 首先调用 `deallocuvm()` 将 0 到 `KERNBASE` 的虚拟地址映射的全部页帧回收，然后销毁整个进程的页表（包括 `PT` 占用的页帧）。`xv6` 的进程 `exit()` 时并不撤销自己的进程内存空间，而是变成`ZOMBIE` 状态后由父进程 `wait()` 时执行 `freevm()` 撤销内存空间和 `PCB`。在 `exec()` 失败时，也会执行 `freevm()` 撤销新创建的进程。

 ## 3. 用户进程空间切换

用户空间切换实际上也伴随着内核空间的切换，因为页表同时覆盖了用户空间和内核空间，
但由于各进程的内核空间映射内容完全相同，因此可以在各进程的内核代码间平滑切换。 `switchuvm()`（[vm.c#L155](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L155)）完成用户进程空间的切换，从使用 `kpgdir` 页表切换为使用 `proc->pgdir` 页表。`switchuvm()` 并不仅指页表切换，还需要有 TSS（涉及内核栈切换）。 `switchuvm()` 根据传入的 `proc` 结构中的页表和 TSS 等信息，执行将该进程的页表载入 `cr3` 寄存器、切换内核堆栈等操作，从而完成虚拟地址空间环境的切换。`switchuvm()` 仅在从 `scheduler` 切换到进程的执行流，或者修改进程镜像时，才会被调用。 共有有三处：`exec()`、`growproc()` 和 `scheduler()`。

## 4. 内核空间与用户空间交换数据 

当用户进程要读写设备上的数据时，必须提供内核空间与用户空间的拷贝机制。其首要条件就是内核态要能访问到进程用户空间，涉及用户空间到内核空间的地址转换。 在 `vm.c` 中 `uva2ka()` 将一个用户地址转化为内核地址，也就是通过用户地址找到对应的物理地址，然后得出这个物理地址在内核页表中的虚拟地址并返回，`copyout()` 则调用 `uva2ka()` 则拷贝 `p` 地址 `len` 字节到用户地址 `va` 中。

 

