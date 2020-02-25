---
title: 6. 进程状态变化
---

因为资源有限，多个进程同时争夺少数的资源，必然导致进程的执行是走走停停的，为了让进程正常运行，很有必要对进程的状态进行描述、改变和保存。

## 1. 睡眠（阻塞）

首先要知道，如果资源充足，谁愿意装睡呢？进程睡眠的根本原因就是资源不够了。

[sleep()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L415) 函数用于进程睡眠阻塞 。需要传入阻塞队列 `chan` 和自旋锁 `lk`，其中 `chan` 根据事件的不同而不同，例如可以是一个 [buf](https://github.com/professordeng/xv6-expansion/blob/master/buf.h) 缓冲区（在 [iderw()](https://github.com/professordeng/xv6-expansion/blob/master/ide.c#L161) 中）。 也就是说，XV6 并没有使用专门的阻塞队列这样一个数据结构，而是通过将等待相同事件的进程 [proc->chan](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L47) 指向相同的的数据对象（即地址）来识别的。

[proc.c#L438](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L438) 将进程挂入到阻塞队列上，并将状态修改为 `SLEEPING`，通过 `sched()` 切换到其他进程。

如果入参 `lk` 不是 `ptable.lock` 的时候，在修改进程状态之前还要对 `ptable.lock` 进行加锁。此时将同时持有 `lk` 和 `ptable.lock`，因此可以将 `lk` 解锁，此时即使有其他进程尝试 `wakeup(chan)`， 也会因为没有 `ptable.lock` 而无法开展 `wakeup()` 操作，必须等我们将 `ptable.lock` 释放。 

由于 `sleep()` 函数有可能在临界区被调用，所以释放 `lk` 这个步骤是必须的，不然会导致死锁现象。

## 2. 唤醒

如果需要唤醒阻塞队列上的所有进程则使用 [wakeup()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L467) 。它在获得进程 PCB 数组 `ptable.lock` 之后，进一步 `wakeup1()` 来完成唤醒工作。 具体操作是通过扫描所有进程，看是否在指定的阻塞队列 `chan` 上睡眠（状态为 `SLEEPING`）， 如果是则将其状态修改为 `RUNNABLE`（`RUNNABLE` 状态将忽视 `proc->chan`）。 

将 `wakeup` 操作分成两部分的原因是，有时候已经持有了 `ptable.lock` 锁，这时只需要直接调用 `wakeup1()` 即可。

从这里也可以看出，确实不存在睡眠阻塞队列，而仅仅是靠等待相同的事件（地址）来表示。 

## 3. 放弃 CPU

进程需要让出 CPU 时执行 [yield()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L384)。所作操作很简单，就是将状态设置为 `RUNNABLE`（本来是正在执行的 `RUNNING` 状态），然后执行 `sched()` 切换到其他进程。

时钟中断 [clock tick](https://github.com/professordeng/xv6-expansion/blob/sem/trap.c#L103) 处理程序中，将会执行 `yield()` 将本进程让出 CPU，并调用 `sched()` 从而完成进程切换。

## 4. 等待

父进程等待子进程退出时将执行 [wait()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L270)。子进程的查找只能通过遍历所有进程并根据其父进程是否指向自己来判定。对找到的子进程，如果其状态为 `ZOMBIE`（执行了 `exit()` 系统调用之后的状态），则需要对僵尸子进程做最后的撤销工作。

如果没有子进程退出，则通过 [sleep()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L309) 进入睡眠阻塞状态（当子进程退出而执行 `exit()` 时会唤醒父进程）。

## 5. 查看进程信息

如果在 XV6 的 shell 命令行中输入 `Ctrl+p` 将显示当前进程的信息列表。该功能由 [procdump()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L499) 实现。该函数对 `ptable.proc[]` 进行扫描，显示 `used` 的 PCB 的关键信息。 

```bash
1 sleep  init 80103e27 80103ec7 80104879 80105835 8010564f
2 sleep  sh 80103dec 801002ca 80100f9c 80104b62 80104879 80105835 8010564f
```

从代码中可以看出每一行的前三列信息包括：进程号、状态、进程名（可执行文件名），因此上述信息表明有 1 号进程 `init` 处于睡眠 `SLEEPING` 状态，2 号进程 `sh` 处于睡眠 `SLEEPING` 状态。后面的数字是各级调用返回地址（EIP 值），最右边是最底层的函数，如果调用次数多于 10 层则只显示 10 层，不足 10 层按实际调用层数显示。

代码中有一个 [NELEM](https://github.com/professordeng/xv6-expansion/blob/master/defs.h#L189) 宏用于计算一维数组的元素个数。 