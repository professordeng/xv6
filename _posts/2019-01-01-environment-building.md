---
title: 1. 运行环境和系统搭建（实验）
---

xv6 是 MIT 设计的一个教学型操纵系统。xv6 可在 Intel X86 框架上运行，为了方便，建议将 xv6 运行在 QEMU 虚拟机器上，本人的实验环境是 ubuntu 18.04 。

## 1. 安装系统

1. 安装 QEMU 软件。

   ```bash
   sudo apt-get install qemu  # 安装软件
   man qemu-system-i386       # 查看使用说明
   ```

2. 下载 xv6 ，本人学习的版本是基于 X86 框架的 [xv6-rev11](https://github.com/mit-pdos/xv6-public/releases)，如果安装了 `git` 可以直接拉取我的仓库主分支。

   注意：下面的示例是用 SSH 拉取的，普通用户用 HTTPS 拉取我的仓库即可，毕竟没有我的密钥。
   
   ```bash
   git clone git@github.com:professordeng/xv6-expansion.git # 拉取代码
   cd xv6-expansion     # 进入目录
   ```
   
3. 编译运行

   ```bash
   sudo apt-get install gcc   # gcc 负责编译
   make                       # 编译
   make qemu-nox              # 运行
   ```

   运行上面的指令若没有出错就会进入 xv6 系统操作界面，显示信息如下：

   ```bash
   qemu-system-i386 -nographic -drive file=fs.img,index=1,media=disk,format=raw -drive file=xv6.img,index=0,media=disk,format=raw -smp 2 -m 512 
   xv6...
   cpu1: starting 1
   cpu0: starting 0
   sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58
   init: starting sh
   ```

   其中 `qemu-system-i386` 就是模拟下的 X86 平台，`-nographic` 不需要图形设备（对服务器友好），`-smp` 是选择启动 CPU 的核数（2 核），`-m` 是指给此系统分配的内存大小。

4. 使用 QEMU 调试功能

   按 `ctrl + a`，再按 `c`，进入 QEMU 调试功能，包括但不限于以下功能

   - 按 `q` 退出虚拟机器。
   - 输入 `info registers` 查看所有寄存器信息。
   - 按 `ctrl + a`，再按 `c` 又回到 xv6 系统。
   - 输入 `info mem` 显示映射的虚拟地址和权限。
   - 输入 `info pg` 显示当前页表结构。

5. 补充

   xv6 也可以使用 `make qemu` 启动，此时会启动两个窗口。鼠标被捕捉后可以用 `Alt + Ctrl` 组合键解脱。

## 2. 简单使用系统

执行 `make qemu-nox` 进入 xv6 操作系统后，就可以使用系统了。xv6 实现了一小部分 Linux 系统上通用的命令，例如：

```bash
ls                        # 显示当前目录下的文件
cat README.md             # 将 README.md 的内容打印到屏幕上
echo hello                # 输出 hello 到屏幕上
cat README.md | grep qemu # 用管道连接两个命令
wc README.md              # 对 README.md 的内容进行统计
rm wc                     # 删除 wc 文件
```

这是一些常见的命令，还有一些其他命令可用 `ls` 查看。

- 拓展（可以跳过）

  通过 `ctrl + p` 查看进程信息（由 `proc.c` 文件中的 `procdump()` 内核函数实现）。可以看到 `sleep` 进程后面跟着一串数字，是调用栈中关于函数调用的地址。例如

  ```bash
  1 sleep init 80103e27 80103ec7 80104879 80105835 8010564f
  2 sleep sh   80103dec 801002ca 80100f9c 80104b62 80104879 80105835 8010564f
  ```

  打开另一个终端，利用 `addr2line` 工具，执行如下命令

  ```bash
  addr2line -e kernel 80103e27
  ```

  可以知道地址 `0x80103e27` 对应于 `kernel` 内核代码 `proc.c` 文件的第 445 行，显示信息如下：

  ```bash
  /home/ubuntu/xv6-expansion/proc.c:445
  ```

  查看 `proc.c` 文件的第 445 行可知，该行代码位于 `sleep()` 函数中。

## 2. 修改系统代码

接下来做个最简单的热身动作，打开 `main.c` 文件，将 `cprintf()` 函数打印的启动提示信息 `cpu0: starting 0` 修改成 `cpu0: let 0 go`，然后执行下面指令进入系统

```bash
make 
make qemu-nox
```

你会发现系统提示信息已经改变，恭喜你，你开始动手修改 `xv6` 操作系统了。 





