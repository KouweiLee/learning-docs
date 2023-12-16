# Arm体系结构学习

## 基础知识

ARM每个异常等级都有一个专门的SP寄存器。

## 寄存器

零寄存器：64位，XZR；32位，WZR

PSTATE寄存器：表示当前处理器状态

ELR_ELx：exception Link reg，当trap到Elx时，该寄存器保存返回地址

SCTLR_ELx：系统控制寄存器

## A64指令集

### 加载与存储指令

* 内存加载指令

1. LDR，`LDR 目标寄存器, <存储器地址>`

变式：

`LDR Xd, =label`：将lable标记的地址放入Xd中。注意，这个lable是一个链接地址，也就是链接脚本中规定的虚拟地址，不一定是实际运行的地址（例如物理地址）。如果label是一个宏，则会把计算出来宏的值加载进Xd。

2. ADR/ADRP指令

ADR加载一个label地址，ADRP也是，不过它加载label地址时会按4KB大小对齐加载进来。

ADR/ADRP与LDR伪指令有本质区别，虽然都是加载一个lable的地址。LDR加载的是链接地址，而ADR加载的是当前PC的相对地址，也就是实际的运行地址（当未开启虚实地址转换时，ADR加载的就是label所在的物理地址了，而如果开启虚实地址转化怒，链接地址与运行地址相同，那么这两个指令就一样了）

需要注意的是：

ldr到一个64位寄存器，那么地址需要按8字节对齐。

* 内存存储指令：

STR

MOV指令：可用于寄存器之间、寄存器与立即数的搬移。`MOV Xd, Xn`

* 系统寄存器的访问

MSR：系统寄存器<-----通用寄存器

MRS：通用寄存器<-----系统寄存器

### 算术与移位指令

加指令：add，adds（影响C标志位），adc

减指令：sub，subs，sbc

比较指令：

`cmp xn, xm`，常与条件操作后缀连用。cmp也会影响PSTATE中的标志位C，若`xn + NOT(xm) + 1`溢出，则C置位，否则，C不置位。

CMN指令，相当于执行ADDS指令，将一个数与另一个数的相反数进行比较。

CSEL：根据cond将Xn（真）或Xm（假）赋值给Xd

CSET：若cond为真，Xd置为1；否则为0

CSINC：类似于CSEL，不过返回的值是Xm会加1

* 移位指令包括

LSL：逻辑左移，操作数左移，低位补0

LSR：逻辑右移，往右移位，左边补0

ASR：算术右移，往右移位，最高位按照符号进行拓展

ROR：循环右移指令，最低位会移动到最高位

* 位操作指令

与：and，ands（影响Z标志位）

或：orr

异或：eor

位清除：bic

计算寄存器为1的最高位前面有几个0：clz

位段插入指令：BFI，将一个寄存器的段插入到另一个寄存器的某段中

位段提取指令：UBFX、SBFX

### 跳转和返回指令

* 跳转

B label：无条件跳转指令

B.cond label

BL label：类似于B，不过是将返回地址放在x30寄存器

BR Xn

BLR Xn：将返回地址放在x30

* 返回

RET：返回到LR（X30寄存器）的地址中

ERET：首先将SPSR的内容恢复到PSTATE，从ELR中获取返回地址，并跳转

* 比较并跳转指令

CBZ

CBNZ：如果不为0，则挑战

TBZ

TBNZ

### 特权级切换指令

有3种提升特权级指令：

```c
• SVC #imm16 // Supervisor call, allows application program to call the kernel(EL1).
• HVC #imm16 // Hypervisor call, allows OS code to call hypervisor (EL2).
• SMC #imm16 // Secure Monitor call, allows OS or hypervisor to call Secure Monitor (EL3).
```

调用这些指令后，会进入异常处理函数。imm16可以在ISS编码中找到，一般用于调试。

## 汇编语法

本地标签配合b使用时，f表示向前（向下）搜索本地标签，b表示向后（向上）搜索本地标签，最近的标签会被执行。

### 伪指令总结

`.align x`：按$2^x$字节对齐

### 内联汇编



如果指定一个局部变量的寄存器，可以通过以下语法：

```c
register int *foo asm ("r12");
```

这将指定foo对应的寄存器为r12

### 链接器与链接脚本

SECTIONS命令，用来描述输出文件的内存布局。

在链接脚本中定义的符号，只代表一个内存地址，没有为其分配什么内存。

## 异常处理

### 发生异常

当异常发生时，首先会确定目标异常等级，然后CPU会自动做一些事情：

* SPSR_ELx <-- PSTATE. 其中ELx为目标异常等级

* ELR_ELx <-- 返回地址

* 设置PSTATE.DAIF均为1, 即屏蔽所有中断

* 对于同步异常，分析其原因并写入ESR_ELx

* 切换SP寄存器为SP_ELx或SP_EL0

  > 发生异常后，CPU会自动根据SPSel寄存器的SP字段来判断是使用SP_EL0(0)还是SP_ELx(1)，x表示目标异常等级。

* 切换到异常目标等级，跳转到异常向量表

> 目标异常等级的确定：发生异常时，根据SCR_EL3和HCR_EL2寄存器的配置可以确定异常目标等级。一般来讲，发生中断均路由到EL2，普通异常还是在EL1

异常向量表的基地址保存在VBAR_Elx中（Vector Base Address Reg，x表示目标异常等级）。

异常向量表中有多个表项，根据3个属性来选择对应的表项中的指令来执行：

1. 异常类型
2. 异常如果是从当前EL进入到当前EL，以及要使用的SP_EL类型
3. 异常如果是从低等级EL trap到当前EL，低等级的执行状态

### 异常返回

执行`ERET`指令即可从异常返回，该指令的作用是：

* PC <-- ELR_ELx
* PSTATE <-- SPSR_ELx 

因此，ERET是可以直接从EL2返回到EL0的，只要发生异常时的状态为EL0,目标异常等级为EL2.

## 内存管理

分页模式下，虚拟地址64位，若页面大小为4KB，则页内偏移12位，从高到低L0-L3共4级页表，每级占9位，故有效虚拟地址为48位。通过计算，一个L1和L2页表项分别能管理1GB和2MB的内存，故也可以直接指向一个相同大小的块内存。这些话用下图表示就是：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309272219476.png" alt="image-20230927221904377" style="zoom:50%;" />

内存分有2种：

1. 普通类型内存：允许处理器做许多优化，包括高速缓存、乱序加载
2. 设备类型内存：访存时会严格按照指令顺序执行，可以根据是否聚合、指令重排、提前写应答进行更细粒度划分

这两种内存属性存放在MAIR_ELn中，该寄存器分为8段，每段表示不同的内存属性，每个页表项通过AttrIndx[2:0]索引到MAIR_ELn中的某一段，作为该页表项对应页面的内存属性。

* 页表项有一些常见的重要属性：

缓存性和共享性：缓存性其实就是指上面提到的内存属性，一般普通内存都会使能高速缓存。共享性则是使能高速缓存后的附加属性，其中内部共享表示多核处理器中该内存区域的高速缓存只能被具有内部共享属性的高速缓存的CPU观察到，外部共享表示系统中所有的CPU都可观察到，没有共享表示只有本地CPU能观察到

> cpu访存流程：cpu->页表->cache

访问权限和执行权限

* 重要寄存器

TCR：地址转换控制寄存器

SCTLR：打开MMU地址转换、数据和指令高速缓存

TTBR：页表基地址

* 内存屏障指令

DMB：Data Memory Barrier，确保在下一次存储器访问前，存储器访问操作全部完成

DSB：Data Synchronization Barrier，确保在下一个指令执行前，存储器访问全部完成

ISB：指令同步屏障，清空流水线，执行新的指令前，之前所有指令全部完成

## 高速缓存

高速缓存用来解决处理器访问速度和内存访问速度严重不匹配的问题。

处理器在查找cache时，会将地址分割为如下形式：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310111005147.png" alt="image-20231011100536821" style="zoom: 67%;" />

高速缓存，是由n个路组成的。例如32KB大小的4路组相联高速缓存中，就有4个路，每路大小为32/4=8KB。如果cache line的大小为32B，则一路包含8KB/32B=256个cache line。这些cache line的寻址则是通过索引域，此例中256个cache line则需要8位的索引域。

索引域相同的多路cache line可以看成一个**组**。当通过索引域索引到一组cache line后，通过比对各个cache line的标记域(tag)，如果能找到对应的cache line，则命中；否则，则需要访存。

这样，如果索引域相同、但并不相等的多个地址，其数据则会缓存到同组上的不同路。因此cache在查找时，则会并行地根据索引域对所有路进行cache line的查找，并进行标记域的比对。

高速缓存根据是通过物理地址还是虚拟地址寻址，可以分为物理高速缓存和虚拟高速缓存：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310181027250.png" alt="image-20231018102755984" style="zoom: 67%;" />

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310181028969.png" alt="image-20231018102822751" style="zoom:67%;" />

但由于寻址是利用了索引域和标记域2个坐标，因此还可以细分为3种高速缓存：

1. VIVT(Virtual Index Virtual Tag)：使用虚拟地址的索引域和标记域。
2. PIPT：使用物理地址的索引域和标记域
3. VIPT：使用虚拟地址的索引域和物理地址的标记域。这种做法好处有：修改虚拟地址和物理地址映射关系后，不需要invalidate 对应的cache line；当索引域全部位于页内偏移区域时，标记域代表物理地址，标记域就确定了物理页号，不会产生重名问题。

使用虚拟高速缓存时，可能会出现2种问题：

1. 重名：多个不同的虚拟地址映射到同一个物理地址，导致修改一个虚拟地址数据时，可能会导致同一物理地址的缓存数据不一致。
2. 同名：同一个虚拟地址映射到不同的物理地址，例如进程切换时如果不进行一些机制的处理，则可能出现这种问题。

### 高速缓存的策略

当处理器要执行读写内存操作时，这些操作首先会查询cache，然后根据cache设定的不同策略进行以下判断和决策。

#### 写内存

如果写命中，则有策略：

1. 直写：数据同时写入当前和下一级cache line以及主存
2. 回写：数据只写入当前高速缓存，只有当该cache line被替换时，才会更新到下一级cache line或主存

如果写未命中，则有策略：

1. 写分配：分配一个新的cache line并写入数据
2. 不写分配：不分配cache line，直接写入内存

#### 读内存

如果读命中，则直接从cache获取数据；如果读未命中，则有策略：

1. 读分配：先将数据加载到cache，再从中获取数据
2. 读直通：不经过cache，直接从内存读

### 维护高速缓存

cache缓存的是内存里的数据，而内存中的数据能被cache缓存的前提是该内存区域是**可缓存的普通内存**。进一步地，我们还可以指定这个内存区域是内部共享的还是外部共享的，如果是内存共享的，则可以缓存在处理器核内部的cache，如L1 cache；如果是外部共享的，则只能缓存在通过系统总线扩展的高速缓存，可能是L3 cache。

对高速缓存的管理分为3种情况：

1. 失效：使cache失效，丢弃其数据
2. 清理：将某个cache line写回到内存或下一级高速缓存。
3. 清零

通过指令DC、IC可以分别对数据高速缓存和指令高速缓存进行管理。其格式为：

```
DC <operation>, <Xt>
IC <operation>, <Xt>
```

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310181108211.png" alt="image-20231018110813002" style="zoom:80%;" />

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310181110302.png" alt="微信图片_20231018111009" style="zoom: 50%;" />

### 查看高速缓存的信息

CLIDR_EL1寄存器：通过Ctype<n\>字段标识系统最高支持几级高速缓存，通过ICB字段标识最高级别的内部共享的高速缓存是第几级。

CTR_EL0寄存器：IminLine和DminLine分别表示指令高速缓存和数据（以及联合）高速缓存的cache line大小；L1Ip表示L1指令高速缓存的策略，例如虚拟索引物理标记等。

CSSELR_EL1和CSSIDR_EL1寄存器：通过指定CSSELR_EL1寄存器中要查询的cache层级、类型，可以通过CSSIDR_EL1得到对应cache的路、组数量。

## 缓存一致性

缓存一致性关注的是同一个数据在多个高速缓存和内存之间的一致性。对于系统级别的缓存一致性，即CPU簇与簇之间、CPU和GPU之间的缓存一致性，可以通过CCI、CCN等控制器实现，不过如果没有这些控制器，则需要软件实现。我们重点关注CPU簇之内缓存一致性的实现。

通过MESI协议可以解决同簇多核CPU的高速缓存一致性问题。其实现方式基于，多核处理器系统中，cache维护指令会通过总线广播机制发给所有CPU内核。

MESI协议规定每个cache line的状态为：修改M、独占E、共享S、无效I 4个状态中的一个。通过cache line上的标志——脏、有效和共享3位组合可以表示这4个状态。每个cache都会监听其他高速缓存的总线活动，从而实现缓存一致性。

* 高速缓存伪共享

若多个处理器反复读写一个cache line上的不同数据，例如数组中两个相邻的元素，那么根据MESI协议会不断地使对方的cache line失效，不断地将数据写入内存，导致性能大大下降，这种现象叫做高速缓存伪共享。

解决办法：让多线程操作的数据处在不同的高速缓存行，这可以通过cache line对齐和填充实现。

## TLB

TLB是MMU转换虚拟地址到物理地址的一个缓冲区，由一些表项组成。表项的结构为：

| VPN  | PPN  | ASID | 其他属性 |
| ---- | ---- | ---- | -------- |

每个CPU核都会有一个TLB，一般TLB也分2级，L1级TLB包含指令TLB和数据TLB，L2级则不区分。

TLB类似于高速缓存的形式，一般采用组相联形式，VPN被分为了标记域和索引域。

启用ASID功能时，切换进程不需要invalidate所有TLB表项，只需要在找到符合当前VPN的TLB表项后，比对这些表项的ASID和TTBR的ASID，TTBR的ASID指示了当前进程的ASID和页表基地址，相同的则表示命中。

另外对于虚拟化，TLB中还会增加VMID域，这样切换虚拟机时就不需要Invalidate整个TLB。

### tlb维护指令

tlb维护指令用于刷新tlb：

```
tlbi 参数
```

典型的有：

```ass
tlbi allen # 将eln中当前CPU整个tlb失效
```

如果想保证tlbi指令的执行顺序，在单处理机系统中需要在其后加入`dsb nsh`保证tlbi指令执行完；多处理机系统则需要加`dsb ish`保证内部共享域的所有CPU都执行完tlbi。

## GIC

GIC：Generic Interrupt Controller

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548428.png" alt="image-20230924154311191" style="zoom:50%;" />

外设发给GIC的中断分为两种类型：信号或message（MSI，通过写GIC的一个寄存器）。

GIC通过mpidr来确定中断要到达哪个核。

安全模型：每个INTID都需要被指定一个group，GICv3支持3种group：Group 0，Secure Group 1和Non-Secure Group 1。Group 0总是会成为FIQ，Group 1则根据当前安全状态和异常等级成为FIQ或IRQ：

![image-20230930214749228](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309302147422.png)

根据SCR_EL3中IRQ和FIQ的设置，若IRQ为0，表示所有IRQ中断不在EL3处理，FIQ为1，表示所有FIQ中断都到EL3处理，这样就有了下面这张图：

![image-20230930215730376](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309302157492.png)

* 编程模型

GICv3的寄存器接口可以分为3个部分：

1. Distributor interface

   Distributor寄存器是MMIO的，被用来配置SPI

2. Redistributor interface

   每个core有一个Redistributor，被用来配置SGI和PPI

3. CPU interface

   用来handle interrupt，是一组系统寄存器，命名为**ICC\_*\_ELn**

   使用前需要enable ICC_SRE_ELn SRE bit。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548457.png" alt="image-20230924171239029" style="zoom:50%;" />

* GIC可以支持多种中断：

Shared Peripheral Interrupt（SPI)：外设中断可以被传送到任意一个core

Private Peripheral Interrupt (PPI)：外设中断private to one core。

Software Generated Interrupt，通常用于核间通信，可以通过写SGI寄存器来产生该中断。

INTID表：

![image-20230925224635370](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309252246571.png)

![image-20230925224657137](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309252246214.png)

* GIC的状态模型

GIC维护着一个状态自动机，包含4个状态：

Inactive. The interrupt source is not currently asserted. 

Pending. The interrupt source has been asserted, but the interrupt has not yet been acknowledged by a PE.

Active. The interrupt source has been asserted, and the interrupt has been acknowledged by a PE. 

Active and Pending. An instance of the interrupt has been acknowledged, and another instance is now pending.

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548447.png" alt="image-20230924214554557" style="zoom:33%;" />

一个中断的生命周期，取决于它是level-sensitive还是edge-triggered。对于level-sensitive，外设需要一直发出中断信号，直到被取消；对于edge-triggered，只需要发一下就行。

**level sensitive interrupts：**

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548504.png" alt="image-20230924215203463" style="zoom:33%;" />

Inactive->Pending：中断源发出中断，GIC发出信号给PE

Pending->AP:当PE读CPU interface中IAR寄存器承认中断，这样GIC会停止中断信号。

AP->Active：当软件写入外设的状态寄存器，表示正式处理该中断。

Active->Inactive：当PE写EOIR寄存器时，表示PE已经完成对该中断的处理。

**Edge-triggered interrupts：**

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548407.png" alt="image-20230924220041938" style="zoom:33%;" />

和Level-sensitive不同之处主要在于：

Active->AP：如果外设再次发出中断信号，则该中断在GIC中的状态转为AP

AP->Pending：当PE写EOIR寄存器表示完成中断处理后，GIC再次发出中断信号给PE，等待其处理

### 配置和使用GIC

GIC的设置分为两种：全局和针对不同中断类型，先来看全局：

#### 全局设置

* GICD_CRLR：Distributor control register

ARE：是否处于v3还是v2模式

* GICD_CTLR：enable 不同Group的中断分发
* GICR_WAKER：记录PE是否是online。
* SCR_EL3、HCR_EL2 ：决定中断发生后交由哪个EL解决

#### 不同中断类型

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548988.png" style="zoom:50%;" />

* Priority: GICD_IPRIORITYn, GICR_IPRIORITYn

这两个寄存器用于定义每个INTID的优先级，值越小优先级越高

* Group: GICD_IGROUPn, GICD_IGRPMODn, GICR_IGROUPn, GICR_IGRPMODn

指定特定的中断类型是属于哪个group

* Edge-triggered or level-sensitive: GICD_ICFGRn, GICR_ICFGRn

* Enable: GICD_ISENABLERn, GICD_ICENABLER, GICR_ISENABLERn, GICR_ICENABLERn

### Running priority and preemption

running priority是指PE的优先级，用于确定是否可以抢占。当PE承认中断时，这个中断的优先级首先必须大于priority mask寄存器规定的最小优先级，此时running priority会变为该中断的优先级，中断结束后会恢复原状。running priority的值记录在ICC_RPR_EL1

当PE正在处理一个中断时，发生了另一个中断，如果该中断的Group  priority大于当前中断的，则发生抢占。ICC_BPRn_EL1规定了priority多少位属于Group priority，例如8bit priority：

![image-20231001081424673](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310010814789.png)

### 中断处理

当一个中断源发出中断时，从GIC到PE处理共分为几步：

1. routing pending interrupt to PE

i. 检查INTID所属的Group是否在distributor和CPU interface中的寄存器中enable了。如果没有，则中断一直处于pending状态。

ii. 检查INTID本身是否enable

iii. 确定要到达的PE：

对于SPI，由GICD_IROUTERn决定。每个SPI中断都有一个对应的IROUTER，其指示了SPI中断对应的PE。这个PE可以是一个指定的，也可以是系统中所有的PE。

对于LPI，由ITS决定

对于SGI，则由SGI寄存器中的Target List决定

iv. ICC_PMR_EL1记录了该PE能接受的最小优先级，只有比它高的PE才能接收该中断。

v. 检查running priority，检查如果PE正在处理中断，判断Group priority是否可以抢占。

如果这些检查都通过了，那么PE就会收到IRQ或FIQ异常。

2. taking an interrupt

PE进入异常处理函数时，要根据IARs（ Interrupt Acknowledge）获取INTID。有2种IARs:

![image-20230925110035576](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548155.png)

读取IAR可能也会返回一些保留值，根据这些保留值可以获取下一步要做什么，比如从一种执行状态转换到另一种，这时读取IAR不会承认中断。

3. end of interrupt

中断完成后，软件需要通知GIC，分为2步

i. priority drop：恢复running priority为中断以前的值。

ii. Deactivation：更新状态机从Active-> inactive

ICC_CTLR_ELn.EOImode决定这两步是分开进行还是同时进行：

ROImode==0：当写 ICC_EOIR0_EL1取消Group 0中断，或写 ICC_EOIR1_EL1结束Group 1中断时，priority drop和deactivation会同时进行。

EOImode==1：当写以上两个寄存器时，仅发生priority drop。再写ICC_DIR_EL1才会deactivation。

另外可能要检查几个点：

i. 检查最高优先级的pending interrupt和running priority

ICC_HPPIR0_EL1、ICC_HPPIR1_EL1：report the INTID of the highest priority pending interrupt for a PE.

ICC_RPR_EL1：Running Priority register

ii. 检查特定中断的状态：

GICD_ISACTIVERn：读取SPI是否被激活。也可用于激活SPI，在测试中断是否有效时可以使用 

GICD_ICACTIVERn： 读取SPI是否被激活。也可用于失效SPI。

GICR_ISACTIVERn：用于SGI和PPI

GICR_ICACTIVERn：用于SGI和PPI

### SGI

软件写以下3个SGI寄存器之一即可触发SGI中断：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548289.png" alt="image-20230924220550256" style="zoom:50%;" />

SGI寄存器基本格式如下：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309251548351.png" alt="image-20230924220701300" style="zoom:50%;" />

SGI ID：INTID号

IRM：0表示中断发送给`aff3.2.1.<Target List>`中，`<Target List>`相当于一个mask对aff0；1表示中断发送给所有连接的PE（除了自己）

### Redistributor register map

每个redistributor定义了两个64KB的物理页帧：

1. RD_base
2. SGI_base：for controlling and generating PPIs and SGIs

### vGIC学习

vPE：虚拟机运行时的PE，一个物理pPE上可以虚拟出多个vPE。对于jailhouse，由于PE是分割的，所以vPE就是pPE

vGIC主要是针对CPU interface的虚拟化，CPU interface的寄存器可以分为3组：

![image-20230927212119833](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309272121052.png)

* Physical CPU interface registers：hypervisor通过这些寄存器处理物理中断

* Virtualization control registers：用于控制虚拟化的feature，例如是否启动vCPU interface等。

* Virtual CPU interface registers：guest OS通过使用`ICV_*_EL1`寄存器来处理虚拟中断。这些寄存器的功能和作用与同名的`ICC_*_EL1`完全相同。其实读取ICV和ICC在汇编码中是一样的，在EL2和EL3，CPU会认为都是ICC寄存器。在EL1，HCR_EL2寄存器中的routing bits决定是ICC还是ICV寄存器被读取。

  vCPU interface 寄存器分为3组：

  1. Group 0 Registers：解决Group 0中断，例如ICC_IAR0_EL1 and ICV_IAR0_EL1，当HCR_EL2.FMO==1，EL1中选择的是ICV寄存器
  2. Group 1 Registers：解决Group 1中断，例如ICC_IAR1_EL1and ICV_IAR1_EL1，当HCR_EL2.IMO==1，EL1选择ICV寄存器
  3. Common Registers：用于解决Group 0和1的中断，例如ICC_DIR_EL1 and ICV_DIR_EL1，当HCR_EL2.IMO\==1 **or** HCR_EL2.FMO==1，EL1选择ICV寄存器。

#### 管理虚拟中断

Hypervisor使用List registers `ICH_LR<n>_EL2`，可以向vPE发出虚拟中断。每个寄存器代表一个虚拟中断，并包含如下信息：

虚拟环境下中断的ID：vINTID、虚拟中断的状态（自动更新）。

List Register的格式：

![image-20231002093320936](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310020933089.png)

Priority：由ICH_VTR_EL2指示有多少priority bits

### el1下的中断处理

当采用虚拟化时，el1下GIC发出中断到指定CPU，如果此时CPU的daif寄存器没有屏蔽所有中断，则会跳到EL2对应的异常处理函数进行处理。

## CPU管理

PE：Processing Element，独立计算单元，一般指一个CPU core

SMP：指这个系统中各个CPU是平等的，同构的。

cluster：一个多核处理器中，多个核被外界可以看做是一个cluster或者叫一个unit

MPIDR_EL1：Multi-Processor Affinity Register，可以让软件知道自己运行在哪个核上。其域中Aff(3, 2, 1, 0)可以指示core的编号，以一个8核2 cluster 非超线程cpu为例， core0的mpidr_el1的affinity为（0，0，0，0），core1为（0，0，0，1）,以次类推， core7则为（0，0，1，3）。Arm规范要求了每个core的（Aff3，Aff2,Aff1,Aff0）编码必须唯一。

![image-20230924150634661](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309241506005.png)

## 问题

- [ ] sbc考虑的C是计算之前的C，还是计算之后的C
- [x] enable mmu时，会自动invalidate掉cache吗？不会，需要自己执行一系列指令invalidate掉所有指令高速缓存
