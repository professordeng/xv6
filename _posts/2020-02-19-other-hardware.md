---
title: 2. 其他硬件
date: 2019-10-03
---

## 1. 定时器

`xv6` 使用的是 8253 定时器，在内核初始化的时候调用了 `timerinit()`，该函数的功能就是让 8253 定时器每秒产生 100 次时钟中断。

`TIMER` 定义在 [lapic.c#L32](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L32)。

## 2. IDE 磁盘 / 块设备

早期的硬盘 IDE（Integrate Device Electronics）的读取方式比较复杂，需要分别指定 CHS （cylinder/head/sector，即柱面/磁头/扇区）；后来的磁盘都支持 LBA（logical block addressing，逻辑块寻址）模式，所有扇区都统一编号，只需指出扇区号即可完成访问。

下面介绍在 LBA 的模式下的 PIO（program IO）来实现对磁盘的读取操作。主板有两个 IDE 通道，每个通道可挂载两个硬盘。访问第一个通道的第一个硬盘的扇区使用 IO 地址寄存器（0x1f0~0x1f7）；访问第二个硬盘使用的是（0x1f8~0x1ff）。访问第二个通道的硬盘的硬盘分别使用（0x170-0x177）和（0x178-0x17f）。我们以第一个通道的第一个硬盘为例，说明 IO 端口寄存器的作用（注意选取本通道的主/从硬盘由第六个寄存器决定）：

1. 0x1f0：读数据，当 0x1f7 不为忙状态时，可以读。 
2. 0x1f2：每次读扇区的数目（最小是 1）。
3. 0x1f3：如果是 LBA 模式就是 LBA 参数的 0-7 位（相当于扇区）。
4. 0x1f4：如果是 LBA 模式就是 LBA 参数的 8-15 位（相当于磁头）。
5. 0x1f5：如果是 LBA 模式就是了 LBA 参数的 16-23 位（相当于柱面）
6. 0x1f6：第七位必须 1，第六位 1 为 LBA 模式 0 为 chs 模式，第五位必须 1，第四位 是 0 为主盘、1 为从盘，3-0 位是 LBA 的参数 27-24 位。
7. 0x1f7：读该寄存器将获得状态信息，写出寄存器则可以发出命令（例如发出读命令，然后在不是忙状态的下可以从 0x1f0 读取数据）。命令举例如下：20H 读扇区，出错时允许重试（重读次数由制造商确定），2lH 禁止重试；30H 写扇区（允许重试）， 31H 禁止重试。

对照上述定义，我们查看系统启动时载入内核的代码，[bootmain.c#L58](https://github.com/professordeng/xv6-expansion/blob/master/bootmain.c#L58) 的 `readsect()` 函数，可以看出 `0x1f2=1` 标识读入一个扇区，`0x1f3~0x1f5` 给出的是 LBA 的第 0~23 位，0x1f6 为 0xE0（0x1110-0000）标识 LBA 模式访问第一个 IDE 控制器上的第一块（编号为 0）的硬盘， 且 LBA 的第 24~27 位为 0x0000。然后才通过 `outb(0x1F7, 0x20)` 发出 0x20 的读命令，并通过 `waitdisk()` 等待磁盘空闲，最后通过 `insl(0x1F0, dst, SECTSIZE/4)` 读入整个扇区的数据，由于一次读入四字节，因此使用 SECTSIZE/4 次 IO 操作。

[ide.c](https://github.com/professordeng/xv6-expansion/blob/master/ide.c)

注意，IDE 驱动程序中并没有使用 DMA 方式，有兴趣的读者可以将代码修改为支持 DMA 方式以提升系统效率。

### 3. 中断控制器

如果系统中只有一个 CPU，那么可以通过主 PIC 芯片直接连接到 CPU 的 INTR 引脚，再配上一个 PIC 芯片可以将外设中断扩展到 15 个。但是对于多 CPU 系统来说，上述方案并不能工作，我们需要能将中断传到每个 CPU 的能力，这时需要更复杂的 APIC。

### 3.1 APIC

只讨论软件，硬件只需要讲清楚 IOPIC 会将中断信号通过消息传递给 LAPIC，后面就是 CPU 的事了，不属于 IO 部分了。

 **xv6 中的 APIC**

对于多核环境，`xv6` 使用 IO APIC 来接收外设中断事件，而用 LAPIC（Local APIC）将中断事件传递到处理器。时钟芯片位于 LAPIC，因此每个处理器核可以接收到独立的时钟中断，`xv6` 在 [lapicinit()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L54) 中完成时钟芯片的设置，将时钟中断路由到 IRQ0（IRQ_TIMER），对应的中断向量是 32。需要区分 PIC 的中断引脚线和 IDT 表项的编号关系，IRQ0 对应到 IDT 表的 32 项。注意比较 IDT 表的第 64 项是用于系统调用的，因此使用的是 trap  gate（不关中断，不清除 IF 标志位），而 IDT 表的 32 项是中断门（关中断，清除 IF 标志位）。

处理器标志寄存器 `eflags` 的 IF 可以屏蔽外设终端，`cli` 指令实现该功能，`sti` 作用相反。启 动时处理器禁止接收中断，在 `scheduler()` 中打开中断。在特定的代码片段，例如 `switchuvm()` 会短暂关闭中断。 

APIC 机制包括外设端的 IOAPIC 和各个处理器核的 LAPIC 两部分组成。当 IO device 通过 IOAPIC 的 interrupt lines 产生 interrupt，IOAPIC 将根据内部的 PRT table 格式化成中断请求消息，并将该消息发送给目标 CPU 的 LAPIC，再由 LAPIC 通知 CPU 进行处理。

Intel APIC 由一组中断输入信号，一个 `24*64` bit 的 Programmable Redirection Table(PRT)， 一组 register 和用于向 APIC BUS（FSB/QPI） 总线上传送 APIC MSG 消息的部件组成。 

**APIC中的中断处理过程 **

1. 外设： 外设发出中断到 IOAPIC 引脚。

2. IOAPIC

   IOAPIC 会查询引脚对应的 RTE。

   IOAPIC 根据 RTE 的设定，决定是否 mask 该中断。

   IOAPIC 根据 RTE 的设定，如 deliver mode, vector 等构建 interrupt message IOAPIC 将 interrupt message 发送给 LAPIC 

3. LAPIC

   LAPIC 接收 interrupt message，设定 IRR ISR 等。 

   LAPIC 提取 interrupt message 中的 vector，交给 processor。

4. CPU

   processor 查 IDT，完成中断处理 

   processor 发送 EOI 

5. LAPIC→IOAPIC→外设完成中断完成动作 

**LAPIC**

收到来自 IOAPIC 的中断消息后，LAPIC 会将该中断交由 CPU 处理。和 IOAPIC 比较， LAPIC 具有更多的寄存器以及更复杂的机制。几个重要的寄存器： 

