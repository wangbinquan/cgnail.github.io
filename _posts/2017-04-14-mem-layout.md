---
layout: post
title: "计算机中的内存"
modified:
categories: academic
excerpt:
tags: ["翻译", "内存系列"]
---

>进一步了解有关内存的诸多信息，参阅Ulrich Drepper所写的一篇经典文章《[What Every Programmer Should Know About Memory](http://people.redhat.com/drepper/cpumemory.pdf)》。
>
>C语言将内存的管理和使用交给了程序员，因此很容易导致内存错误。而内存错误的发生点，在时间和空间上可能和错误源相距甚远，于是很难发现。《深入理解计算机系统》的9.11节描述了[C语言中常见的内存错误](../mem-err-in-c)。

## CPU与内存映射

对于CPU来说，它一点也不知道它连接了什么东西。CPU仅仅通过一组针脚与外界交互，它并不关心外界到底有什么。CPU主要通过3种方式与外界交互：内存地址空间，I/O地址空间，还有中断。

前端总线把CPU与北桥连接起来（北桥则连接RAM内存、各种PCI设备，以及南桥）。每当CPU需要读写内存时，都会使用这条总线。一个Intel Core 2 QX6600有33个针脚用于传输物理内存地址（可以表示2^33个地址位置），64个针脚用于接收/发送数据（所以数据在64位通道中传输，也就是8字节的数据块）。这使得CPU可以控制64GB的物理内存（2^33个地址乘以8字节），尽管大多数的芯片组只能支持8GB的RAM。

当北桥接收到一个物理内存访问请求时，它查找内存地址映射表以决定将请求发给哪个设备。内存地址映射表知道每一个物理内存地址区域所对应的设备。一个典型的Intel系统的低端4G内存布局如下图所示，其中，棕色的区域都被设备地址映射走了，这些被映射为设备的内存地址形成了一个经典的空洞，位于PC内存的640KB到1MB之间。当内存地址被保留用于显卡和PCI设备时，就会形成更大的空洞。这就是为什么32位的操作系统无法使用全部的4GB RAM。

![Intel系统中，低端4GB内存地址空间的布局](/images/mem-layout.png)

记住，这些在主板总线上使用的都是**物理地址**。在CPU内部（比如我们正在编写和运行的程序），使用的是**逻辑地址**，必须先由CPU翻译成物理地址以后，才能发布到总线上去访问内存。

CPU的运行模式（实模式，32位保护模式，64位保护模式）决定了物理->逻辑地址的翻译规则

* 实模式，只能寻址1MB的物理地址空间
* 32位保护模式，可以寻址4GB物理地址空间。由于顶部的大约1GB物理地址被映射到了主板上的设备，CPU实际能够使用的也就只有大约3GB的RAM
* 64位保护模式，可以寻址64GB的地址空间，虽然很少有芯片组支持这么大的RAM

## 内存地址转换与分段

x86 CPU开启分页功能后的内存地址转换过程如下图所示。

![x86保护模式下的寻址](/images/mem-addr-trans.PNG)

CPU先通过分段机制将代码中的逻辑地址翻译成**线性地址**，再利用分页机制将线性地址翻译成物理地址，交给北桥寻址。

### 分段

16位长的段寄存器（segment register）在实模式和保护模式下运作方式并不相同。

* 实模式下，采用SegReg*16+off的方式寻址
* 保护模式下，段寄存器保存的项目叫段选择符。其中高13位保存了段描述符表的表项索引。每条索引对应一个8字节（64bit）长的段描述符。段描述符包含了段基地址、段界限、特权级等信息。第2位（TI位）区别GDT(TI=0)/LDT(TI=1)，剩下的两位称为请求特权级（RPL）。

段描述符表有两个，分别是全局描述符表（GDT）和局部描述符表（LDT）。电脑中的每一个CPU（或一个处理核心）都含有一个叫做gdtr的寄存器，用于保存GDT的首个字节所在的线性内存地址。也就是说，每个CPU一份GDT。在[系统的引导启动过程](../bootstrap)中head.S会初始化GDT，IDT和二级页表，它们放在物理内存从地址0x0开始的地方。

对于一条指令的寻址，过程如下：

1. 从对应的段选择符（如代码段是CS，数据段是DS），读出对应段索引
2. 段索引加上gdtr的基地址，获得段描述符
3. 从段描述符中得到基地址，加上指令中给出的偏移量，得到线性地址

然而，在32位保护模式下，（32位）寄存器和指令（的32位操作数）都可以寻址整个线性地址空间，所以Linux/Windows实际上采用了Intel的“扁平模型”，关闭了分段功能，让逻辑地址与线性地址一致。

在Linux上，只有3个段描述符在[系统的引导启动过程](../bootstrap)中被使用。其中两个段是扁平的，可对整个32位空间寻址：一个是代码段，加载到cs中，一个是数据段，加载到其他段寄存器中。第三个段是系统段，称为任务状态段（Task State Segment）。在完成引导启动以后，Linux设置了4个主要的GDT表项，即此时系统内存管理分为4个段：内核代码段，内核数据段，用户代码段，用户数据段。其中，2个内核段是扁平的（CS,DS的值是0xC00000000），另两个用于用户模式（CS,DS的值是0x00000000）。也就是说，对于上述寻址过程来说，用户模式下段描述符的基地址均为0.

经典的Unix错误信息“Segmentation fault”（分段错误）并不是由x86风格的段所引起的，而是由于分页单元检测到了非法的内存地址（参见[C语言中常见的内存错误](../mem-err-in-c)）。

### 分页

386及以上的x86处理器使用内存分页机制，这使得同一个线性地址可以被映射为多个物理地址，这种映射是通过（MMU中的）分页单元这一特殊的硬件电路实现的。操作系统通过维护每个进程私有的页目录和页表实现线性地址与物理地址之间的转换。

页是逻辑意义上的结构，它是具有固定大小的一组连续线性地址的集合，每一个逻辑页在页表中都有一个与之对应的32位的页表项（page table entry，PTE）进行描述。而每个页实际对应的物理内存则被称为页框（page frame）。目前Windows和Linux均采用4K大小的页和页框，也就是说，一个逻辑页正好映射一个物理页框。

对于32位机器，可寻址空间为4G，每个页框4K，于是需要1M个PTE，占据4MByte空间。为了节省空间，对4G空间采用两级分页机制：第一级为1K个页目录表（Page Directory Table），通过它保存的页目录项（Page Directory Entry）找到第二级的页表，每个页表保存1K个页表项（PTE），对应每个页。这样两级表分别占据4KByte空间，正好一个页框，描述全部4G内存总共只需要2个页框。

处理器中与分页单元有关的寄存器为CR0~CR3控制寄存器，其中CR1被处理器保留，CR2寄存器则用于存放页故障线性地址，当根据某个线性地址所寻址的页不在内存中时将触发一个缺页异常，此时处理器负责将该线性地址加载至CR2寄存器从而把适当的页重新加载到内存中。与分页单元联系最紧密的当属CR0和CR3控制寄存器。

CR0中保存了

* PG位，分页机制的开关，关闭时线性地址和物理地址一一对应。
* PE位，用于实模式与保护模式之间的转换，只有保护模式下才能打开分页机制。
* CD位，用于启用或禁用CPU中的L1、L2高速缓存。
* 写时拷贝位WP、等等

CR3则保存页目录基地址，每次进程切换都会更新之。于是，寻址过程如下：

1. CR3+虚拟地址高10位（31~22）得到页目录项PDE
2. 从PDE中读出的页表基地址+虚拟地址中间10位（21～12）得到PTE
3. 从PTE中读出的页的基地址+虚拟地址最后12位（11～0）得到具体物理地址

这一过程由？？完成。显然寻址是很频繁的过程，而内存的访问速度远不如CPU和总线的传输速度，于是引入了各式缓存：

1. L1/L2。L1在CPU内，细分为指令和数据，L2在CPU外，不细分。由CR0中的CD位控制开启
2. 转移后备缓冲区（ Translation Lookaside Buffer，TLB）。在CPU的MMU（x86构架下MMU位于CPU中，但其它构架则不一定，如ARM的MMU作为协处理器存在，此外，ARM没有段寄存器，是真正扁平的CPU）中。存放物理地址以加速对线性地址的转换操作。当CR3更新时（进城切换），TLB也会无效。

## 内存的保护

CPU特权级总共有4个，编号从0（最高特权）到3（最低特权）。有3种主要的资源受到保护：内存，I/O端口以及执行特殊机器指令的能力。通常，x86内核只用到其中的2个特权级：0和3。

CPU指令中只有大约15条指令被限制只能在ring 0执行（其余那么多指令的操作数都受到一定的限制）。这些指令如果被用户模式的程序所使用，就会颠覆保护机制或引起混乱，导致一个一般保护错（general-protection exception）。

前面说到段寄存器可以保存段选择符，而段选择符包含了请求特权级（Requested Privilege Level，简称RPL）字段，用于控制权限。实际上，数据段选择符可由程序直接加载到各个段寄存器当中，比如ss（堆栈段寄存器）和ds（数据段寄存器），但代码段寄存器（cs）不行。cs只能被那些会改变程序执行顺序的指令（如CALL）间接的设置，而且，cs拥有一个由CPU自己维护的当前特权级字段（Current Privilege Level，简称CPL），字段的值总是等于CPU的当前特权级。也就是说，在任何时候，不管CPU内部正在发生什么，只要看一眼cs中的CPL，你就可以知道此刻的特权级了。

由于限制了对内存和I/O端口的访问，用户模式代码在不调用系统内核的情况下，几乎**不能**与外部世界交互。这就从设计上就决定了：进程所使用的资源会在进程结束后被统统回收和清理。所有的控制着内存或打开的文件等的数据结构全都不能被用户代码直接使用；一旦进程结束了，这些数据会被内核销毁。于是，只要硬件和内核不出毛病，系统可以一直不当机。这也解释了为什么Windows 95/98那么容易死机：这并非因为微软差劲，而是因为系统中的一些重要数据结构，出于兼容的目的被设计成可以由用户直接访问了。

CPU会在两个关键点上保护内存：当一个段选择符被加载时，以及，当通过线形地址访问一个内存页时。当一个数据段选择符被加载时，CPU挑出CPL和RPL中特权最低的一个，并与描述符特权级（descriptor privilege level，简称DPL）比较，如果DPL的值大于等于它，段描述符对应的内存就可以访问。其设计思想是：允许内核代码加载特权较低的段。但堆栈段寄存器是个例外，它的加载要求CPL，RPL和DPL这3个值必须完全一致。

事实上，段保护功能几乎没什么用，因为现代的内核使用扁平的地址空间。在那里，用户模式的段可以访问整个线形地址空间。真正有用的内存保护发生在分页单元中，即从线形地址转化为物理地址的时候。一个内存页就是由一个页表项（page table entry）所描述的字节块。页表项包含两个与保护有关的字段：一个超级用户标志（supervisor flag），一个读写标志（read/write flag）。前者开启时该页不能被ring 3访问，后者则负责赋予页的读写权限，以防止一些指针错误。

最后，我们需要一种方式来让CPU切换它的特权级。这项工作是通过门描述符（gate descriptor）和sysenter指令来完成的。一个门描述符就是一个系统类型的段描述符（也包含DPL），分为了4个子类型：

1. 调用门描述符（call-gate descriptor），提供了一个可以用于通常的CALL和JMP指令的内核入口点，用得不多。
2. 任务门描述符（task-gate descriptor），用得不多，在Linux上，它们只在处理内核或硬件问题引起的双重故障时才被用到。
3. 中断门描述符（interrupt-gate descriptor），
4. 陷阱门描述符（trap-gate descriptor）

中断门和陷阱门用来处理硬件中断（如键盘，计时器，磁盘）和异常（如缺页异常，0除数异常）。他们被存储在中断描述符表（Interrupt Descriptor Table，简称IDT）当中。每一个中断都分配一个[0, 255]的中断向量作为IDT索引，用来指出当中断发生时使用哪一个门描述符来处理中断。

内核一般在门描述符中填入内核代码段的段选择符，这样结合偏移量（Offset）可以寻址中断处理代码的入口点。一个中断只会将控制权限保持（被内核中断）或提升（被用户中断）。无论哪一种情况，作为结果的CPL必须等于目的代码段的DPL。如果CPL发生了改变，一个堆栈切换操作就会发生。如果中断是被程序中的指令所触发的，门的DPL>=CPL，以防用户代码随意触发中断。

要进入ring 0，可以调用sysenter指令启动系统调用。此时，，CPU不再进行特权检查，而是直接进入CPL 0，并将新值加载到与代码和堆栈有关的寄存器当中（cs，eip，ss和esp）。当需要跳转回ring 3时，内核发出一个iret或sysexit指令，分别用于从中断和系统调用中返回，从而离开ring 0并恢复CPL=3的用户代码的执行。

