---
title: 6. 中断控制器
---

如果系统中只有一个 CPU，那么可以通过主 PIC 芯片直接连接到 CPU 的 INTR 引脚，再配上一个 PIC 芯片可以将外设中断扩展到 15 个。但是对于多 CPU 系统来说，上述方案并不能工作，我们需要能将中断传到每个 CPU 的能力，这时需要更复杂的 APIC。

## 1. APIC

只讨论软件，硬件只需要讲清楚 IOPIC 会将中断信号通过消息传递给 LAPIC，后面就是 CPU 的事了，不属于 IO 部分了。

 ### 1.1 xv6 中的 APIC

对于多核环境，XV6 使用 IOAPIC 来接收外设中断事件，而用 LAPIC（Local APIC）将中断事件传递到处理器。时钟芯片位于 LAPIC，因此每个处理器核可以接收到独立的时钟中断，XV6 在 [lapicinit()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L54) 中完成时钟芯片的设置，将时钟中断路由到 [IRQ0](https://github.com/professordeng/xv6-expansion/blob/master/traps.h#L30)（IRQ_TIMER），对应的中断向量是 32。需要区分 PIC 的中断引脚线和 IDT 表项的编号关系，IRQ0 对应到 IDT 表的 32 项。注意比较 IDT 表的第 64 项是用于系统调用的，因此使用的是 trap  gate（不关中断，不清除 IF 标志位），而 IDT 表的 32 项是中断门（关中断，清除 IF 标志位）。

处理器标志寄存器 `eflags` 的 IF 可以屏蔽外设终端，[cli](https://github.com/professordeng/xv6-expansion/blob/master/x86.h#L108) 指令实现该功能，`sti` 作用相反。启 动时处理器禁止接收中断，在 [scheduler()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L331) 中打开中断。在特定的代码片段，例如 [switchuvm()](https://github.com/professordeng/xv6-expansion/blob/master/vm.c#L155) 会短暂关闭中断。 

APIC 机制包括外设端的 IOAPIC 和各个处理器核的 LAPIC 两部分组成。当南桥的 IO device 通过 IOAPIC 的 interrupt lines 产生 interrupt，IOAPIC 将根据内部的 PRT table 格式化成中断请求消息，并将该消息发送给目标 CPU 的 LAPIC，再由 LAPIC 通知 CPU 进行处理。

Intel APIC 由一组中断输入信号，一个 `24*64` bit 的 Programmable Redirection Table（PRT），一组 registers 和用于向 APIC BUS（FSB / QPI） 总线上传送 APIC MSG 消息的部件组成。 

### 1.2 APIC 中的中断处理过程 

1. 外设： 外设发出中断到 IOAPIC 引脚。

2. IOAPIC

   IOAPIC 会查询引脚对应的 RTE。

   IOAPIC 根据 RTE 的设定，决定是否 mask 该中断。

   IOAPIC 根据 RTE 的设定，如 deliver mode，vector 等构建 interrupt message IOAPIC 将 interrupt message 发送给 LAPIC 

3. LAPIC

   LAPIC 接收 interrupt message，设定 IRR ISR 等。 

   LAPIC 提取 interrupt message 中的 vector，交给 processor。

4. CPU

   processor 查 IDT，完成中断处理 

   processor 发送 EOI 

5. LAPIC→IOAPIC→外设完成中断完成动作 

### 1.3 LAPIC

收到来自 IOAPIC 的中断消息后，LAPIC 会将该中断交由 CPU 处理。和 IOAPIC 比较， LAPIC 具有更多的寄存器以及更复杂的机制。几个重要的寄存器： 

In-Service Register（ISR），256 bit（`8*64`），每个 bit 对应一种外设中断。

Interrupt Request Register（IRR），256 bit（`8*64`），每个 bit 对应一种外设中断。

EOI Register

Task Priority Register（TPR），每个 LAPIC 会被分配一个仲裁优先级（0-15），低于该优先级的中断会被 APIC 屏蔽。

Processor Priority Register（PPR），处理器优先级寄存器，取值范围为 0~15。该寄存器决定当前 CPU 正在处理的中断的优先级级别，以确定一个 Pending 在 IRR 上的中断是否发送给 CPU。 与 TPR 不同，它的值由 CPU 动态设置而不是由软件设置，因此 XV6 代码不设置该寄存器。

Interrupt Command Register（ICR），例如要发 IPI 消息给其他 CPU 核。

Local Vector Table（LVT）。本地中断向量表。

对于处理来自 IOAPIC 的中断消息，最重要的寄存器还是 IRR、ISR 以及 EOI。 

IRR（Interrupt Request Register）：功能和 PIC 的类似，代表 LAPIC 已接收中断，但还未交 CPU 处理。 

ISR（In-Service Register）：功能和 PIC 类似，代表 CPU 已开始处理中断，但还未完成。 与 PIC 有所不同的是，当 CPU 正在处理某中断时，同类型中断如果发生，相应的 IRR bit 会再次置一（PIC 模式下，同类型的中断被屏蔽）；如果某中断被 pending 在 IRR 中，同类型的中断发生，则 ISR 中相应的 bit 被置一。这说明在 APIC 系统中，同一类型中断最多可以被计数两次（超过两次时，不同架构处理不一样）。对于 Pentium 系列 CPU 和 P6 架构，中断消息被 LAPIC 拒绝；对于 Pentium4 和 Xeon 系列，新来的中断和 IRR 中对应的 bit 重叠。

**中断优先级**

中断的优先级别由下列公式：优先级别 = vector / 16

外设的 32~255 号 vector 构成了 2~15 共 14 个优先级别。对于同一个级别的中断，vector 号越大的优先级越高。例如 vector33、34 都属于级别 2，34 的优先级就比 33 高。所以，对于 8 bit 的 vector，又可以划分成两部分，高 4 bit 表示中断优先级别，低 4 bit 表示该中断在这一级别中的位置。

TPR 的值增加 1，将会屏蔽 16 个 vector 对应的中断。NMI、SMI、ExtINT、INIT、start-up delivery 的中断不受 TPR 约束。XV6 在 [lapicinit()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L96) 中设置 TPR=0，表示 APIC 不屏蔽任何外设中断（仍可能被 CPU 屏蔽）。 

### 1.4 中断来源

对于目前的 LAPIC 来说，它可能从以下几个来源接收到中断： 

1. Locally connected IO devices：这个主要是指通过 local interrupt pins（LINT0 and LINT1）直接和处理器相连的 IO 设备。
2. APIC timer generated interrupts ：LAPIC 可以通过编程设置某个 counter，在固定时间内向处理器发送中断。
3. Performance monitoring counter interrupts ：这个是指处理器中的性能计数器在发生 overflow 事件的时候向处理器发送中断进行通知。
4. Thermal Sensor interrupts ：这个是由温度传感器触发的中断。
5. APIC internal error interrupts：这个是 LAPIC 内部错误触发的中断。
6. Externally connected IO devices：这个是指外部设备通过 IOAPIC 和 LAPIC 进行相连。
7. Inter-processor interrupts（IPIs）：这个是指处理器之间通过 IPI 的方式进行中断的发送和接收。

其中，前面五种中断来源被称为本地中断源（local interrupt sources），LAPIC 会预先在 Local Vector Table（LVT）表中设置好相应的中断递送（delivery）方案，在接收到这些本地中断源的时 候根据 LVT 中的方案对相关中断进行递送。 

除此之外，对于从 IOAPIC 中发送过来的外部中断，以及从其它处理器中发过来的 IPI 中断，LAPIC 会直接将该中断交给本地的处理器进行处理。而如果需要向其它处理器发送 IPI，则可以通过写 LAPIC 中的 ICR 寄存器完成。 

### 1.5 LVT

LVT 实际上是一片连续的地址空间，每 32-bit 一项，作为各个本地中断源的 APIC register ： 

bit 0-7： Vector，即 CPU 收到的中断向量号，其中 0-15 号被视为非法，会产生一个 Illegal Vector 错误（即 [ESR](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L83) 的 bit 6）。

bit 8-10：Delivery Mode，有以下几种取值： 

000 (Fixed)：按 Vector 的值向 CPU 发送相应的中断向量号。 

010 (SMI)：向 CPU 发送一个 SMI，此模式下 Vector 必须为 0。

100 (NMI)：向 CPU 发送一个 NMI，此时 Vector 会被忽略。

101 (INIT)：向 CPU 发送一个 INIT，此模式下 Vector 必须为 0。

111 (ExtINT)：令 CPU 按照响应外部 8259A 的方式响应中断，这将会引起一个 INTA 周期， CPU 在该周期向外部控制器索取 Vector。APIC 只支持一个 ExtINT 中断源，整个系统中应当只 有一个 CPU 的其中一个 LVT 表项配置为 ExtINT 模式。

bit 12：Delivery Status（只读），取 0 表示空闲，取 1 表示 CPU 尚未接受该中断（尚未 EOI）。

bit 13：Interrupt Input Pin Polarity，取 0 表示 active high，取 1 表示 active low。 

bit 14：Remote IRR Flag（只读），若当前接受的中断为 fixed mode 且是 level triggered 的， 则该位为 1 表示 CPU 已经接受中断（已将中断加入 IRR），但尚未进行 EOI。CPU 执行 EOI 后， 该位就恢复到 0。

bit 15：Trigger Mode，取 0 表示 edge triggered，取 1 表示 level triggered。

bit 16：为 Mask，取 0 表示允许接受中断，取 1 表示禁止，reset 后初始值为 1。

bit 17/17-18：Timer Mode，只有 LVT Timer Register 有用，用于切换 APIC Timer 的三种模式。

最后两种中断通过写 ICR 来发送。当对 ICR 进行写入时，将产生 interrupt message 并通过 system bus（Pentium 4 / Intel Xeon）或 APIC bus（Pentium / P6 family）送达目标 LAPIC 。 

当有多个 APIC 向通过 system bus / APIC bus 发送 message 时，需要进行仲裁。每个 LAPIC 会被分配一个仲裁优先级（0-15），优先级最高的拿到 bus，从而能够发送消息。 在消息发送完成后，刚刚发送消息的 LAPIC 的仲裁优先级会被设置为 0，其他的 LAPIC 会加 1。

XV6 在 [lapicinit()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L71) 中将 [LINT0](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L36)、LINT1、ERROR、PC 的 LVT 项都做了设置。

### 1.6 CPU 处理 LAPIC 上的中断 

LAPIC 无论是接收到来自 IOAPIC 的中断，来自本地中断源的中断，还是来自其他处理器发送的 IPI 中断，都会将其交由 CPU 进行处理，但是由于 CPU 这个时候可能正在处理其它中断，所以需要一套机制来保证中断处理的安全性。 

首先需要注意的是，在 RTE 格式那张表中，中断的 delivery mode 可能有好几种，其中 NMI、 SMI、INIT、ExtINT 和 SIPI 这几种 delivery mode 的中断将会直接交由 CPU 进行处理，如果当前 CPU 正在处理这些 delivery mode 的中断，则会禁止相同的中断被递送进来。除此之外，还有 一种被称为 fixed 的 delivery mode，也就是普通的中断，它们的递送机制是通过 IRR 和 ISR 寄存器完成的。在 X86 平台上，这两个都是 256 bits 的寄存器（其实是由 8 个 64 bits 的寄存器组成的），每个 bit 代表一个中断的 vector，其中第 0 到第 16 个 bit 是 reserve 的。IRR 和 ISR 每个 bit 代表的意思分别如下：

1. IRR：如果第 n 位的被置 1，则代表 LAPIC 已接收 vector 为 n 的中断，但还未交 CPU 处理。
2. ISR：如果第 n 位的 bit 被置上，则代表 CPU 已开始处理 vector 为 n 的中断，但还未完成。

需要注意的是，当 CPU 正在处理某中断时，如果又被递送过来一个相同 vector 的中断， 则相应的 IRR bit 会再次置一； 如果某中断被 pending 在 IRR 中，同类型的被再次递送过来， 则 ISR 中相应的 bit 被置一。 这说明在 APIC 系统中，同一类型中断最多可以被计数两次。 

另外，当某个中断被处理完之后，LAPIC 需要通过软件写 EOI 寄存器来告知。

因此，根据处理器的不同，一个典型的 LAPIC 中断处理流程是这样的： 

对于 Pentium4 和 Xeon 系列：

1. 通过中断消息的 destination field 字段，确定该中断是否是发送给自己的。
2. 如果该中断的 delivery mode 为 NMI、SMI、INIT、ExtINT、SIPI，直接交由 CPU 处理。
3. 如果不为以上所列举的中断，则将 IRR 中相应的 bit 置一。
4. 当中断被 pending 到 IRR 寄存器中后，根据 TPR 和 PPR 寄存器，判断当前最高优先级 的中断是否能发送给 CPU 进行处理，并将 ISR 寄存器中相应的 bit 置一。 
5. 软件写 EOI 寄存器通知中断处理完成。如果中断为 level 触发，该 EOI 广播到所有 IOAPIC， NMI、SMI、INIT、ExtINT、SIPI 类型中断无需写 EOI。 

对于 Pentium 系列和 P6 架构：

1. 确定该中断是否由自己接收。如果是一个 IPI，且 delivery mode 为 lowest priority，LAPIC 与其它 LAPIC 一起仲裁该 IPI 由谁接收。 
2. 若该中断由自己接收，且类型为 NMI、SMI、INIT、ExtINIT、INIT-deassert、或 MP 协议 中的 IPI 中断（BIPI、FIPI、SIPI），直接交由 CPU 处理。 
3. 将中断 pending 到 IRR 或 ISR 寄存器，若已有相同的的中断 pending 在 IRR 和 ISR 寄存器上，拒绝该中断消息，并通知 sender “retry”。
4. 同 Pentium4、Xeon 系列流程。 
5. 同 Pentium4、Xeon 系列流程。 

在上面的这两套流程中，涉及到几个关键的寄存器（TPR，PRR）和 delivery mode（lowest priority），这就涉及到中断的优先级问题了，会在 “中断的优先级问题” 中进行解释。 

当 CPU 开始处理中断的时候，会查询一个被称为中断描述符表（Interrupt Descriptor Table， IDT）的数据结构，该数据结构的每一项都被预先填上了一个门描述符（gate descriptor），其中有三种门描述符：task，interrupt 和 trap，这里我们主要关注的是 interrupt-gate descriptor。 

通过它，就可以找到相应 vector 的中断的处理函数了。在进入处理函数之前，一般会对栈进行一个切换，并且将相应的寄存器信息（包括 RFLAGS，CS，RIP 等）压入栈中，从而保证在中断处理结束之后可以恢复相关信息。

主要包括两种情况，第一种情况是被中断的进程不是内核进程，则需要有一个权限级别的切换，因此需要换一个栈；第二种情况是被中断的进程是一个内核进程，因此不需要切换栈， 只需要在原来的栈中保存信息就可以了。整个流程还是比较清楚的，因此这里也不详述了。

### 1.7 相关代码

[lapicw()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L46) 实现写 Local APIC 寄存器，此函数有两个参数，第一个参数为 LAPIC 的偏移地址，第 二个参数为要写入的值。

[cpuid()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L29) 返回正在运行的 CPU 的 ID。

[lapiceoi()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L108) 响应中断，即向 EOI 寄存器发送 0； 

[lapicinit()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L54) 初始化本 CPU 的 Local APIC； 

[lapicstartap()](https://github.com/professordeng/xv6-expansion/blob/master/lapic.c#L126) 通过写 ICR 寄存器的方式启动 AP，此函数有两个参数，第一个参数为要启动的 AP 的 ID，第二个参数为启动代码的物理地址。

## 2. IOAPIC

IOAPIC 接受外设产生的中断，并决定如何、向哪个 CPU 发送该中断。

APIC 和早期的 PIC 项比较的话，最大的差别就是中断请求信号不再是通过硬连线传递，而是通过总线消息的方式传递。IOAPIC 最大的作用在于中断分发。根据其内部的 PRT （Programmable Redirection Table）表，IOAPIC 可以格式化出一条中断消息，发送给某个 CPU 的 LAPIC，由 LAPIC 通知 CPU 进行处理。目前典型的 IOAPIC 具有 24 个中断管脚，每个管脚对应一个 RTE（Redirection Table Entry，PRT 表项）。与 PIC 不同的是，IOAPIC 的管脚没有优先级， 也就是说，连接在管脚上的设备是平等的。但这并不意味着 APIC 系统中没有硬件优先级。设备的中断优先级由它对应的 vector 决定，APIC 将优先级控制的功能放到了 LAPIC 中，我们在后面会看到。 

当 IOAPIC 某个管脚接收到中断信号后，会根据该管脚对应的 RTE，按照规定的格式化构造出一条中断消息，发送给某个 CPU 的 LAPIC。从上表我们可以看出，RTE 给出了一个中断的所有信息。 

IOAPIC 对外只表现为两个地址，分别是内部寄存器选择、读写数据。

经过这个接口，可以访问 IOAPIC 内部的寄存器。

IOAPIC 内部寄存器中的 0x10~0x3F 是 RET 表，用于设定各个外设中断如何向 CPU 传递，传送方式由对应的 8 个字节 RTE 决定。需要注意的是一个 RTE 对应的中断，其中断向量号是通过 `RTE[7:0]` 设置的，起点 [T_IRQ0](https://github.com/professordeng/xv6-expansion/blob/master/traps.h#L30) 定义为 32。还要注意的是 `RTE[63:56]` 是该中断信号应当传递到哪个 CPU 上（可以用 APIC ID 或 CPU ID）。中断传递模式是 `RTE[10:8]` 共有 8 种， `XV6` 外设使用的是 000，类似于老式 PID 的方式（记录在 APIO 的 IRR 和 ISR 寄存器对应的位上）。

### 2.1 相关代码

阅读 [ioapic.c](https://github.com/professordeng/xv6-expansion/blob/master/ioapic.c) 必须结合前面给出的寄存器地址、作用和位域的定义。[IOAPIC](https://github.com/professordeng/xv6-expansion/blob/master/ioapic.c#L9) 宏给出了 IOAPIC 的物理地址 `0xFEC00000`（由于 QEMU 仿真的主板只有一个 IOAPIC，则 `0xFEC0xy00` 中的 `xy` 取值为 00），并将 IOAPIC 的 IO 地址空间用 [ioapic](https://github.com/professordeng/xv6-expansion/blob/master/ioapic.c#L27) 结构体来表示。[ioapicread()](https://github.com/professordeng/xv6-expansion/blob/master/ioapic.c#L34) 和 [ioapicwrite()](https://github.com/professordeng/xv6-expansion/blob/master/ioapic.c#L41) 工作原理相同，通过 `ioapic->reg = reg` 指定寄存器，然后对 `ioapic->data` 进行读写，即可完成对应寄存器的读写操作。

第 `i` 个 RTE 的 64 位信息分成两个 32 位整数来表示，对应两个地址：`REG_TABLE+2*i` 和 `REG_TABLE+2*i+1`。系统刚启动时，所有中断都被设置为边沿触发、高电平有效、屏蔽、不传送状态。在各个设备（例如 IDE 硬盘）初始化时，才将对应的中断通过 [ioapicenable()](https://github.com/professordeng/xv6-expansion/blob/master/ioapic.c#L67) 传递给指定 CPU 核，如下：

```bash
➜  xv6-expansion git:(rawdisk) grep ioapicenable *.c
console.c:  ioapicenable(IRQ_KBD, 0);
ide.c:  ioapicenable(IRQ_IDE, ncpu - 1);
ioapic.c:ioapicenable(int irq, int cpunum)
uart.c:  ioapicenable(IRQ_COM1, 0);
```

可以看出，这些设备中断被 XV6 固定地传送到编号为 0 或 `ncpu-1` 的处理器上，其他处理器将不会处理外设中断。

 