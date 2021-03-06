---
title: 2. 实现简单的用户程序（实验）
---

完成了 XV6 操作系统的搭建后，我们可以利用 XV6 操作系统提供的系统调用来实现一个简单的用户程序。

## 1. 磁盘影像文件的生成

想要给 XV6 添加新的可执行文件，首先需要了解 XV6 磁盘文件系统是如何生成的，然后才能编写应用程序并出现在 XV6 的磁盘文件系统中。XV6 文件系统上的可执行文件的生成包括 2 个步骤：

1. 生成各个应用程序。
2. 将应用程序构成文件系统影像。

首先在 `Makefile` 中有一个默认规则，那就是所有的 `.c` 文件都需要通过默认的编译命令生成 `.o` 文件。另外在 `Makefile` 中有一个规则用于指出可执行文件的生成，内容如下：

#### code-2-1

```makefile
ULIB = ulib.o usys.o printf.o umalloc.o

_%: %.o $(ULIB)
    $(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $@ $^
    $(OBJDUMP) -S $@ > $*.asm
    $(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym
```

该模式规则说明，对于所有的 `%.o` 文件将结合 `$(ULIB)` 一起生成可执行文件 `_%`，比如说 `ls.o`（即依赖文件 `$^`）将链接生成 `_ls`（即输出目标 `$@`）。`-Ttext 0` 用于指出代码存放在 0 地址开始的地方，`-e main` 参数指出以 `main` 函数代码作为运行时的第一条指令，`-N` 参数用于指出 `data` 节与 `text` 节都是可读可写、不需要在页边界对齐。

---

生成 `_%` 文件后，开始创建磁盘文件系统，`Makefile` 中相关的代码如下

```makefile
UPROGS=\
    _cat\
    _echo\
    _forktest\
    _grep\
    _init\
    _kill\
    _ln\
    _ls\
    _mkdir\
    _rm\
    _sh\
    _stressfs\
    _usertests\
    _wc\
    _zombie\

fs.img: mkfs README.md $(UPROGS)
    ./mkfs fs.img README.md $(UPROGS)
```

其中变量 `UPROGS` 包含了所有相关的可执行文件名。磁盘文件系统 `fs.img` 目标依赖于 `UPROGS` 变量，并且将它们和 `README.md` 文件一起通过 `mkfs` 程序转换成文件系统。

## 2. 简单输出

下面我们实现一个简单的用户程序实现 `hello world!` 的输出。创建 `hello.c` 文件，此处的 `printf()` 的第一个参数是文件描述符，用于指出输出文件，例如 0 号是标准输入文件，1 号是标准输出文件，2 号是出错文件。

```c
#include "types.h"
#include "stat.h"
#include "user.h"

int main(){
	printf(1, "hello world!\n"); // 1 是文件描述符，后面会提到
	exit();
}
```

然后修改 `Makefile`，在 `UPROGS` 变量的 `_wc\` 的下一行添加 `_hello\`，然后执行 `make qemu-nox`。查看前 5 行信息如下：

```bash
gcc -fno-pic -static -fno-builtin -fno-strict-aliasing -O2 -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer -fno-stack-protector -fno-pie -no-pie   -c -o hello.o hello.c
ld -m    elf_i386 -N -e main -Ttext 0 -o _hello hello.o ulib.o usys.o printf.o umalloc.o
objdump -S _hello > hello.asm
objdump -t _hello | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > hello.sym
./mkfs fs.img README.md _cat _echo _forktest _grep _init _kill _ln _ls _mkdir _rm _sh _stressfs _usertests _wc _hello _zombie 
```

第一行 `gcc` 编译命令是由默认规则触发的，生成 `hello.o`。第二、三、四行的命令是 [code-2-1](#code-2-1) 的规则触发的，其中 `ld` 链接命令生成 `_hello` 可执行文件。由于可执行文件发生了更新，因此触发了磁盘文件系统 `fs.img` 目标的规则（第五行显示的命令使用 `mkfs` 程序生成 `fs.img` 磁盘影像文件），其中可执行文件列表的倒数第二个就是我们刚生成的 `_hello`。

进入 XV6 后利用 `ls` 查看会多一个叫 `hello` 的应用程序，直接输入 `hello` 执行，可输出我们期待的 `hello world!`。

退出 XV6，在终端执行 `ll _hello`，显示 `_hello` 文件的相关信息如下：

```bash
-rwxrwxr-x 1 ubuntu ubuntu 13K Jan 19 01:04 _hello
```

## 3. 实现 copy 功能

在 XV6 中可以调用系统函数可实现 Linux 自带的 `cp` 命令（说明 `cp` 命令属于用户程序）。代码如下：

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"

char buf[512];

int
main(int argc, char *argv[])
{
  int fd0, fd1, n;

  if(argc <= 2){
    printf(1, "Need 2 arguments!\n");
    exit();
  }
  if((fd0 = open(argv[1], O_RDONLY)) < 0){
    printf(1, "cp: cannot open %s\n", argv[1]);
    exit();
  }
  if((fd1 = open(argv[2], O_CREATE|O_RDWR)) < 0){
    printf(1, "cp: cannot open %s\n", argv[2]); 
    exit();
  }  
  while ((n = read(fd0, buf, sizeof(buf))) > 0){
    write(fd1, buf, n);
  }
  close(fd0);
  close(fd1);
  exit();
}
```

修改 `Makefile` ，然后 `make qemu-nox`，执行 `cp README.md counterpart`，会发现一个名为 `counterpart` 的文件，利用 `cat` 读取其内容，发现和 `README.md` 的内容一模一样。