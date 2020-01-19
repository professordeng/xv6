---
title: 3. 新增系统调用
date: 2019-04-05
---

如果我们修改了 `xv6` 的代码，例如增加了优先级调度，那么就需要有设置优先级的系统调用，并且通过应用程序调用该系统调用进行优先级设置。因此我们需要学习如何增加新的系统调用，以及如何在应用程序中进行系统调用这样才能验证 `xv6` 修改后的功能。

## 1. 系统调用示例 

`xv6` 中普通用户可以用的系统调用都在 `user.h` 中有声明。我们以获取进程号的 `getpid()` 系统调用为例，新建 `printpid.c`，内容如下：

```c
#include "types.h"
#include "stat.h"
#include "user.h"

int 
main(){
	printf(1, "my pid is %d\n", getpid());
	exit();
}
```

添加 `_printpid` 到 `Makefile` 中，执行 `make qemu-nox` 进入 `xv6`。然后执行 `printpid`，就可以看到进程 `printpid` 的进程号。

## 2. 添加系统调用

系统调用涉及较多内容，分散在多个文件中，包括系统调用号的分配、系统调用的分发代码（依据系统调用好）修改、系统调用功能的编码实现、用户库头文件修改等。另外还涉及验证用的样例程序，用于校验该系统调用的功能。下面我们一步一步实现

我们希望进程能知道自己所在的处理器编号，这可以通过一个新的系统调用 `getcpuid()`来实现。步骤如下：

### 2.1 增加系统调用号

`xv6` 的系统调用都有一个唯一编号，定义在 `syscall.h`。我们可以在 `SYS_close` 的后面新增一行内容：

```c
#define SYS_getcpuid 22
```

这里的编号 22 可以是其他值，只要不是前面使用过的就好。

### 2.2 增加用户态入口

为了让用户态代码能进行系统调用，需要提供用户态入口函数 `getcpuid()` 和相关的头文件。

1. 修改 `user.h`

   为了让应用程序能调用用户态入口函数 `getcpuid()`，需要在代码 `user.h` 中加入一行函数原型声明 

   ```c
   int getcupid(void);
   ```

   该头文件应该被应用程序段源代码所使用，因为它声明了所有用户态函数的原型。除此之外所有标准 C 语言库的函数都不能使用，因为 `Makefile` 用参数 `-nostdinc` 禁止使用 `Linux` 系统的头文件，而且用 `-I	.` 指出在当前目录中搜索头文件。也就是说 `xv6` 系统中，并没有实现标准的 C 语言库。

2. `usys.S` 中定义用户态入口

   定义了 `getcpuid()` 原型之后，还需要实现 `getcpuid()` 函数。我们在 `usys.S` 中加入一行（可插入到 28 行后面）：

   ```c
   SYSCALL(getcpuid)
   ```

   `SYSCALL` 是一个宏，定义于 `usys.S` 的第一行。`SYSCALL(getcpuid)` 将把 `getcpuid` 定义为函数入口，然后把 `SYS_getcpuid=22` 作为系统调用号保存到 `eax` 寄存器中，然后发出 `int` 指令进行系统调用 `int $SYS_SYSCALL`。这样进入到系统调用公共入口后，以 `eax` 作为下标在系统调用表 `syscalls[]` 中找到需要执行的具体代码。

   这里定义的 `getcpuid()` 函数，就是用户态在希望执行系统调用时所调用的函数，使得用户代码无需编写汇编指令来执行 `int` 指令。

### 2.3 修改跳转表

在系统调用公共入口 `syscall()` 中，`xv6` 将根据系统调用号进行分发处理。负责分发处理的函数 `syscall()`（定义于文件 `syscall.c`），分发依据是一个跳转表。我们需要这个修改跳转表，首先要在`syscall.c` 第 107 行中的分发函数表 `syscalls[]` 中加入

```c
[SYS_getcpuid]   sys_getcpuid
```

也就是下标 22 对应的是 `sys_getcpuid()` 函数地址（后面我们会实现该函数）。其次，由于 `sys_getcpuid` 未声明，因此要在它前面（例如第 105 行后面的位置）加入如下内容用于指出该函数是外部符号。 

```c
extern int sys_getcpuid();
```

前面提到：当用户发出 22 号系统调用是通过 `getcpuid()` 完成的，其中系统调用号 22 是保存在寄存器 `eax` 的。因此 `syscall()` 系统调用入口代码可以通过 `proc->tf->eax` 获得该系统调用号，并保存在 `num` 变量中，于是 `syscalls[num]` 就是 `syscalls[22]` 也就是 `sys_getcpuid()`。下面是从 `syscall.c` 中截取的`syscall()` 函数。

```c
void
syscall(void)
{
  int num;
  struct proc *curproc = myproc();

  num = curproc->tf->eax;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    curproc->tf->eax = syscalls[num]();
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            curproc->pid, curproc->name, num);
    curproc->tf->eax = -1;
  }
}
```

### 2.4 实现系统调用功能

前面的工作使得用户可以用 `getcpuid()` 作为系统调用的用户态入口，而且进入系统调用的分发函数 `syscall()` 中也能正确地转入到 `sys_getcpuid()` 函数里，但是我们还未实现 `sys_getcpuid()` 函数。步骤如下

1. 在 `sysproc.c` 中加入系统调用处理函数 `sys_getcpuid()`

   ```c
   int
   sys_getcpuid()
   {
     return getcpuid();
   }
   ```

2. 在 `proc.c` 中实现 `getcpuid()` 函数

   ```c
   int
   getcpuid(){
     return cpunum();
   }
   ```

3. 为了让 `sysproc.c` 中的 `sys_getcpuid()` 能调用 `proc.c` 中的 `getcpuid()`，还需要在 `defs.h` 加入一行（在 112 行后面添加） 

   ```c
   void  getcpuid(void);
   ```

   用作内核态代码调用 `getcpuid()` 时的函数原型。`defs.h` 定义了 `xv6` 几乎所有的内核数据结构和函数，也被几乎所有内核代码所包含。

## 3. 验证新系统调用



