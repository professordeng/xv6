---
title: 3. xv6 优先级调度思路
date: 2019-08-05
---

1. vim param.h

   修改 NCPU 为 2 ，设置 CPU 的核数为 2，每次最多只有两个进程同时在跑，易于观察。

2. vim exec.c

   设置只要是 shell 生成的子进程默认优先级为 3 。

3. vim proc.c

   修改调度函数，让高优先级的程序先执行。

   https://github.com/professordeng/xv6-public/commit/631d2069e1c1ec046021df1c51956ece9b21e252#diff-7425ed078bca65febfb9834a8024f288
   
4. 实践

   ```
   make qemu-nox
   foo 2 0.01&
   nice 3 9
   ps
   nice 4 8
   ps
   ```




## 原理

1. vim main.c

   查看主函数，可以发现最后会调用 mpmain() 函数，查找 mpmain() 函数，发现函数调用了 scheduler() ，也就是我们要找的调度函数。

2. scheduler() 函数

   该函数在 proc.c 文件中，返回值为空。函数里面是一个死循环，每次调度的时候，查找状态为 RUNNABLE 的进程，将找到的第一个 RUNNABLE 设置为 RUNNING。一般的，调度需要以下几个过程

   ```c
   while(true){
   	find_runnable(p);
   	
   }
   ```

3. vim trap.c

   查看该文件你会发现，如果进程号不是 0 并且进程的状态是 RUNNING ，并且达到中断条件，就需要切换进程了。

4. vim proc.c

   查看 yield() 函数，该函数会将 proc 的状态设置为 RUNNABLE，并调用 sched() 函数。

   sched() 函数会将 proc 的 context 存到 CPU 的寄存器中。也就是 PC 指针会指向保存的值。然后设置 proc=0，寻找另一个进程 RUNNING。

5. 修改 scheduler()

   ```
   struct proc *p1;
   struct proc *highP = NULL;
   
   highP = p;
   for(p1 = ptable.proc; p)
   ```

   





