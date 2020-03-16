---
title: 2. 增加读写权限控制（实验）
---

xv6 的文件是由目录显示的，而在进程中想要读取文件，要经过以下几个步骤：

1. 用 `open` 打开文件，会生成一个 `file` 结构体，`file` 结构体中包括读写权限
2. `read` 函数可以读文件，`write` 可以写文件，是否可以读写由 `f->readable` 和 `f->writable` 决定。
3. 在 `inode` 中添加相应的变量，使得读写真正由文件本身控制。 

## 1. 实验步骤

在 xv6 中，表示一个文件的结构体如下：

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE } type; // 文件类型
  int ref;                // reference count
  char readable;          // 读权限
  char writable;          // 写权限
  struct pipe *pipe;      // 管道地址
  struct inode *ip;       // 索引节点地址
  uint off;               // 文件偏移
};
```

关于文件读写的系统调用有如下：

```c
int open(const char*, int);       // 打开文件，flags 指定读/写模式
int read(int, void*, int);        // 从文件中读 n 个字节到 buf
int write(int, const void*, int); // 从 buf 中写 n 个字节到文件
int close(int);                   // 关闭打开的 fd
```

xv6 创建文件时，有 4 种模式：

1. 读 1 写 1，可读可写
2. 读 1 写 0，可读不可写
3. 读 0 写 1，不可读，可写
4. 读 0 写 0，不可读，不可写

创建或使用文件的第一步是用 [open()](https://github.com/professordeng/xv6-expansion/blob/dev/sysfile.c#L286) 函数打开对应的文件，`open(pathname, omode)` 实现逻辑如下：

1. 如果 `omode` 包含了 `O_CREATE`，则 `open` 函数会先调用 `create()` 函数创建一个文件。如果不包含 `O_CREATE` 参数，则根据路径名找文件。
2. 如果 1 成功了，那么接下来就分配文件结构体 `f` 对文件的访问权限进行控制，而文件结构体的初始化，则是由 `omode` 来完成，也就是说，进程想怎么来就怎么来，索引节点没有做任何权限处理，这里的 `readable` 和 `writable` 完全是由进程创建文件时指定的。
3. `f->readable` 是设定比较难懂，`omode` 与 `O_WRONLY` 相与，如果 `omode` 包含 `O_WRONLY` 的话，那么得到的结果非零，取非后得到零。也就是说如果是只读，`f-readable` 为零，没毛病。
4. `f->writable` 的值和 `O_WRONLY`、`O_RDWR` 有关，其中一个不为零即可。
5. 这里可以发现 `O_RDONLY` 并没有用到。

可以看出 xv6 在创建文件时需要指定文件的读写权限，由于 xv6 是单用户系统，所以不需要考虑用户之间的权限管理。

所以我们测试一下如下特性。

1. 如果用只读模式创建文件，可否写入数据。
2. 如果用只写模式打开文件，可否读入数据。

源文件 `ftest.c` 内容如下：

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"

int
main()
{
  char str[20] = "hello world!\n";
  int fd = open("content", O_CREATE | O_RDWR);
  int n = write(fd, str, 20);
  printf(1, "n = %d\n", n);
  exit();
}
```

这里的 `open` 函数创建了一个文件，模式是可读可写，此时 `n` 为 20，但是我们将 `O_RDWR` 去掉，发现 `n` 为 -1，也就是说，xv6 已经实现了文件的读写保护模式。

xv6 内执行完此程序后，我们运行 `ls`，可以发现多了个 content 文件，利用 `cat` 指令可以得到 content 的内容

```bash
content        2 20 20
$ cat content
hello world!
```

## 2. cat

Linux 有个常用指令 cat，在 xv6 中也实现了，源文件为 `cat.c`，我们可以发现，cat 其实不是内核服务，而是用户程序，通过调用 `open` 函数打开文件，若没有指定只写（`omode=0`），那么就可读，利用 `read` 来读取文件内容，然后用 `write` 函数将内容写到对应的文件中，默认的文件是控制台（`fd=1`），我们可以重定向将内容输出到另一个文件中。

当完成读取任务后，会调用 `close` 函数释放对应的文件描述符。

注意：文件是否存在，不是由文件描述符决定，而是由 `ftable` 决定，也就是是，xv6 系统最后只能有 100 个文件。