# sysHyper学习

## 启动流程

* 说明

./jailhouse是root cell的用户态程序，用来调用内核模块jailhouse.ko，是面向用户的管理接口。

./jailhouse.ko 是加载到root cell的内核模块，提供部分管理功能，通过ioctl操作为用户态提供接口，使用hypercall调用hypervisor提供的接口

./rvmarm.bin 为jailhouse的hypervisor镜像，运行在EL2，提供虚拟化的核心功能，本项目的sysHyper主要就是实现对它的替换

### src/arch/aarch64/entry.rs：

arch_entry函数是driver跳转到的函数，基本和jailhouse的entry.S相同。aarch64.lds则指定了sysHyper的内存布局。最终switch_stack函数跳转到entry函数。CPU在进入el2后，使用的栈指针保存在PerCpu页面顶部。

### main.rs：

1. primary_init_early：

确定虚实地址偏移量PHYS_VIRT_OFFSET，el2的虚实地址的转换是线性映射。

在heap.rs::init初始化堆空间和全局分配器。

初始化FRAME_ALLOCATOR，即物理页框分配器，分配从cpu_data后到hyperviser_end的区域

创建ROOT_CELL，构建root_cell的stage 2页表对应的地址空间，恒等映射。

2. 通过activate函数，设置VTTBR_EL2，
3. gicv3_cpu_init：每个CPU调用
4. activate_vmm：设置VTCR_EL2、HCR_EL2寄存器

## 启动新cell

### create

driver.c：jailhouse_cmd_cell_create

在该函数中，会对新cell的cpu执行cpu_down函数，新cpu会通过SMC进入el2，并执行handle_psci函数。该函数中会执行汇编指令wfi。

> wfi：Wait For Interrupt is a hint instruction that indicates that the PE can enter a low-power state and remain there until a wakeup event occurs. 

hypercall后，依次执行：

arch_handle_exit-->arch_handle_trap-->handle_hvc-->hypercall-->hypervisor_cell_create

该函数会向指定的CPU发送sgi。

指定的CPU会依次执行

arch_handle_exit-->irqchip_handle_irq1-->gicv3_handle_irq_el1-->test_cpu_el1

- [ ] 在test_cpu_el1函数，CPU会reset自己，设置异常返回地址为0x0（TODO：这会发生什么）

### load

### destroy



## 内存管理

```rust
//! Hypervisor Memory Layout
//!
//!     +--------------------------------------+ - HV_BASE: 0xffff_ff00_0000_0000 (lower address)
//!     | HvHeader                             |
//!     +--------------------------------------+
//!     | Text Segment                         |
//!     |                                      |
//!     +--------------------------------------+
//!     | Read-only Data Segment               |
//!     |                                      |
//!     +--------------------------------------+
//!     | Data Segment                         |
//!     |                                      |
//!     +--------------------------------------+
//!     | BSS Segment                          |
//!     | (includes hypervisor heap)           |
//!     |                                      |
//!     +--------------------------------------+ - PER_CPU_ARRAY_PTR (core_end)
//!     |  +--------------------------------+  |
//!     |  | Per-CPU Data 0                 |  |
//!     |  +--------------------------------+  |
//!     |  | Per-CPU Stack 0                |  |
//!     |  +--------------------------------+  | - PER_CPU_ARRAY_PTR + PER_CPU_SIZE
//!     |  | Per-CPU Data 1                 |  |
//!     |  +--------------------------------+  |
//!     |  | Per-CPU Stack 1                |  |
//!     |  +--------------------------------+  |
//!     :  :                                :  :
//!     :  :                                :  :
//!     |  +--------------------------------+  |
//!     |  | Per-CPU Data n-1               |  |
//!     |  +--------------------------------+  |
//!     |  | Per-CPU Stack n-1              |  |
//!     |  +--------------------------------+  | - hv_config_ptr
//!     |  | HvSystemConfig                 |  |
//!     |  | +----------------------------+ |  |
//!     |  | | CellConfigLayout           | |  |
//!     |  | |                            | |  |
//!     |  | +----------------------------+ |  |
//!     |  +--------------------------------+  |
//!     +--------------------------------------| - free_memory_start
//!     |  Dynamic Page Pool                   |
//!     :                                      :
//!     :                                      :
//!     |                                      |
//!     +--------------------------------------+ - hv_end (higher address)
```

MemorySet

```rust
pub type Stage2PageTable = Level4PageTable<GuestPhysAddr, PageTableEntry, S2PTInstr>;
```

HvSystemConfig::get().hypervisor_memory.size的大小，包含了percpu这些。

### 动态内存分配

在sysHyper中，heap是一段在全局数据段中的，用于Vec等需要在堆上分配的数据结构。

而frame_allocator，则是负责分配free_memory_start到hv_end的区域，分配出的Frame在drop时会自动回收，一般用于页表的分配。当MemorySet被drop时，这些页表跟着都会被自动drop。地址映射则不涉及内存分配。

## 异常中断

**由于在HCR_EL2中设置了IMO和FMO，所有中断都会陷入EL2执行。**

## 启动命令

在主文件夹下执行：

```
make run
make monitor
```

进入linux后执行：

```
make scp
```

```
./setup.sh  
./enable.sh  
./jailhouse cell create configs/qemu-arm64-gic-demo.cell
./jailhouse cell load gic-demo configs/gic-demo.bin
./jailhouse cell start gic-demo
```



## 问题

- [ ] CPU进入el2后（除了启动阶段），使用的栈指针为什么就保存在PerCpu的顶部了。

sp应该是根据指定寄存器SPSel对应的字段设置的。

- [x] cache和TLB有什么区别

TLB用来解决从内存中读取页表进行虚拟地址到物理地址转换速度慢问题，保存虚拟地址和物理地址之间的映射关系。

cache则用来解决cpu和读取DRAM速度差异大的问题，保存物理地址和数据的缓存。cpu访问内存都是经过cache的。

- [x] wfi执行后发生中断后是怎么处理的

wfi执行后，发生中断，此时处于EL2，处于屏蔽中断时，CPU会执行wfi的下一条指令直到eret。当返回el1时，由于GIC发送的中断信号尚未de-assert，故跳入异常处理函数，通过读iar寄存器来回应中断，GIC这样就会停止中断信号了

- [x] 多个cpu在从el2返回用户态后，是怎么处理的？

最开始，多个cpu是属于root linux的，因此之前linux干啥，现在这些cpu还是干啥。

- [x] 针对当前实现，destroy的思路	

- [ ] irq mask为什么出现？inject irq是什么意思

- [x] 执行cell create时，第二个SMC是什么？

根据第二个SMC的code来看，属于psci的AFFINITY_INFO操作，用来查询该CPU是否处于poweroff状态。在哪里调用的还不清楚

- [x] 跳板页面的作用是什么？

当开启或关闭虚实地址转换时，需要保持前后地址不变，因此这些代码需要放在跳板页面

