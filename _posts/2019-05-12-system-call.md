---
title: 3. 新增系统调用
date: 2019-04-05
---

`xv6` 中普通用户可以用的系统调用都在 `user.h` 中有声明。我们以获取进程号的 `getpid()` 为例，新建 `getpid.c`。

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

添加 `_getpid` 到 `Makefile` 中，执行 `make qemu-nox` 进入 `xv6`。然后执行 `getpid`，就可以看到进程 `getpid` 的进程号。

## 1. 添加系统调用

以添加获取 CPU 的 ID 为例。多处理器中每一个处理器都有自己的 ID。

1. 修改 `syscall.h`

   `xv6` 每一个系统调用都有一个唯一编号。在 `syscall.h` 中添加

   ```c
   #define SYS_getcpuid  22
   ```

   这里的编号可以是其它值，只要不重复就行。

2. 修改 `syscall.c`

   