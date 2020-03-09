---
title: 7. 缺页中断（实验）
---

操作系统可以与页表硬件一起使用的许多巧妙技巧之一是堆内存的延迟分配。xv6 应用程序使用 `sbrk()` 系统调用向内核请求堆内存。 有些程序分配到内存但从不使用它，例如实现大稀疏数组，因此如果可以将内存分配推迟到进程真正使用的时候，那么将节省很多内存资源。

在内核中，`sbrk()` 分配物理内存并将其映射到进程的虚拟地址空间。 xv6 并没有实现堆内存的惰性分配，`sbrk()` 的函数如下：

```c
int
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
//  if(growproc(n) < 0)
//    return -1;
  return addr;
}
```

这里用到了 `growproc(n)` 给 xv6 分配内存。我们将其注释掉后运行 xv6 并执行 `ls` 指令，得到下面的结果：

```bash
$ ls
pid 3 sh: trap 14 err 6 on cpu 0 eip 0x112c addr 0x4004--kill proc
```

提示信息表明 3 号进程（`sh` 进程），发生了 14 号中断，发生中断的处理器是 0 号处理器，`eip` 的值为 `0x112c`，可根据此内容定位到引起中断的代码，中断时的线性地址为 `0x4004`。

也就是说，访问线性地址 `0x4004` 时发生了缺页异常，发生缺页异常时，会把引起缺页异常的线性地址填到 `cr2` 寄存器中，所以解决这个异常，我们就需要实现相应的异常处理程序。

进入 `trap.c` 文件中，那里有个 `trap` 函数，负责根据中断号来调用对应的异常处理程序，我们将缺页异常的处理函数放到 `switch-case` 语句中，内容如下 

```c
case T_PGFLT:
  pgfault();
  break;
```

接下来在 `vm.c` 中实现 `pgfault()` 函数，内容如下：

```c
void pgfault(){
  char *mem;
  uint a;
  a = PGROUNDDOWN(rcr2());   // cr2 记录引起缺页中断的线性地址
  if(a >= myproc()->sz) {    // 越界
    cprintf("invalid address!\n");
    myproc()->killed = 1;
    return;
  }
  mem = kalloc();            // 此时申请物理页帧
  if (mem == 0) {
    cprintf("kalloc out of memory!\n");
    myproc()->killed = 1;
    return;
  }
  memset(mem, 0, PGSIZE);
  mappages(myproc()->pgdir, (char*)a, PGSIZE, V2P(mem), PTE_W | PTE_U);
  lcr3(V2P(myproc()->pgdir));
}
```

这样，缺页异常被我们实现为了延迟分配的功能。

