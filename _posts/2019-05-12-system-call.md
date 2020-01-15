---
title: 系统调用
---

`xv6` 中用户可以用的系统调用都在 `user.h` 中有声明。我们以获取进程号的 `getpid()` 为例。

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

按照前面的方法修改 `Makefile` 并重新生成 `xv6` 。执行后观察成功打印进程自己的进程号。

## 添加系统调用

以添加获取 CPU 的 ID 为例。多处理器中每一个处理器都有自己的 ID。

1. 修改 `syscall.h`

   `xv6` 每一个系统调用都有一个唯一编号。在 `syscall.h` 中添加

   ```c
   #define SYS_getcpuid  22
   ```

   这里的编号可以是其它值，只要不重复就行。

2. 修改 `syscall.c`

   