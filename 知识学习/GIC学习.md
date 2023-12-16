# GIC-v3学习

## GIC是什么

GIC，是Arm下的一种中断控制器，它可以将外设发来的中断发给特定的CPU。如下图所示，GIC接收n个外设的中断引脚，然后分发给2个CPU：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20231201101135562.png" alt="image-20231201101135562" style="zoom: 33%;" />

因此对于物理板子来讲，外设会发送中断信号给GIC，经由GIC发给CPU；对于qemu，其模拟的virtio-blk就通过GIC发出中断。

## 中断号和中断类型

每个中断通过中断号INTID表示，中断可分为几种类型：

* SGI：Software Generated Interrupt，用于核间通信，通过写GIC中SGI寄存器产生该中断。
* PPI：Private Peripheral Interrupt ：中断只会发给一个指定的核，例如时钟中断。
* SPI：Shared Peripheral Interrupt：中断可以被传送到任意一个CPU，或者所有CPU

![image-20230925224635370](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309252246571.png)

![image-20230925224657137](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309252246214.png)

## GIC把中断发给哪个CPU？

每个CPU核(PE)有一个唯一标识，称为affinity。寄存器MPIDR_EL1记录了该PE的affinity。affinity共32bits，分为4个8 bits的域：

```
<affinity level 3>.<affinity level 2>.<affinity level 1>.<affinity level 0>
```

这些不同level的含义因不同板子而定，但一般是level越大的表示的范围越大，比如level 3表示国家，level 0表示乡镇。

通过affinity和interrupt routing mode（IRM）就可以确定中断发给哪个CPU

### SGI

软件通过写`ICC_SGI0R_EL1`或`ICC_SGI1R_EL1`寄存器来产生Group 0或1中断。寄存器中的IRM bit可以指定是发给所有PE还是affinity对应的PE。

### PPI

感谢神，赐

### SPI

对于INTID为`n`的SPI，`GICD_IROUTER<n>`通过affinity值和IRM指定了目标CPU是哪个或哪些。

## 重要的寄存器

GIC的寄存器接口可分为3类：

* Distributor interface

  用来配置SPI，寄存器名称为`GICD_*`

* Redistributor interface

  配置SGI、PPI，以及电源管理。寄存器名称为`GICR_*`

* CPU interface

  用来处理中断。均为系统寄存器`ICC_*_ELn`

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20231201153858023.png" alt="image-20231201153858023" style="zoom:50%;" />

### 全局设置

GICD_CTLR：enable group 0/1的分发

### 中断配置

为每个中断指定优先级：`GICD_IPRIORITYn`, `GICR_IPRIORITYn`

为每个中断指定Group：`GICD_IGROUPn, GICD_IGRPMODn, GICR_IGROUPn, GICR_IGRPMODn`

为每个PPI和SPI指定是edge-triggered还是level-sensitive：`GICD_ICFGRn, GICR_ICFGRn`。SGI永远是edge-triggered

Enable每个中断发给CPU interface：`GICD_ISENABLERn, GICD_ICENABLERn, GICR_ISENABLERn, GICR_ICENABLERn`。

> ISENABLER只能用于使能中断，写入1表示使能，写入0无效果；ICENABLER只能用于禁止中断，写入1表示禁止，写入0无效果。之所以弄这两种寄存器，是为了替代以前使能或禁用时繁琐的过程，避免出错。

### 承认中断

当PE进入中断的异常处理函数时，就要从`ICC_IAR0_EL1`和`ICC_IAR1_EL1`读取INTID，这样就承认了中断。IAR0表示Group 0的中断（通常用于FIQ），IAR1表示Group 1的中断（通常用于IRQ）。

## 中断虚拟化

GIC的中断虚拟化是指对CPU interface的虚拟化，可以产生一个目标为vPE的虚拟中断。

* 什么是虚拟中断？

虚拟中断是hypervisor发给vPE的一种中断，虚拟中断号的含义和物理中断号完全相同，会发给指定的vPE。vPE会把虚拟中断当成真实的中断处理。

* 为什么需要虚拟中断？

当VM运行在hypervisor上时，VM自己认为自己跑在真实硬件上，但其实发给它的所有中断都会陷入EL2处理（HCR_EL2.IMO/FMO设置的）。但这样对于像时钟中断需要EL1处理的情况下，就需要通过虚拟中断注入到VM中，让VM来处理。

* 如何产生虚拟中断？

通过写List registers `ICH_LR<n>_EL2`，指定虚拟中断号，即可产生虚拟中断。虚拟中断的目标CPU，和真实中断如何确定目标CPU相同。

### 重要寄存器

CPU interfaces虚拟化后，寄存器可分为3组：

* ICC: Physical CPU interface registers
* ICH: Virtualization control registers
* ICV: Virtual CPU interface registers

重要的寄存器如下：

`ICH_LR<n>_EL2`：List寄存器，格式如下：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310020933089.png" alt="image-20231002093320936" style="zoom: 50%;" />

* State：该虚拟中断当前的状态
* HW：该虚拟中断是否对应一个真实的物理中断。如果是，当deactivate该虚拟中断时，也会deactivate pINTID对应的物理中断

`ich_elrsr_el2`：其低16位中，第n位表示List寄存器`ICH_LR<n>_EL2`的状态。若为1，表示该寄存器空闲，可以用来发出虚拟中断；若为0, 表示该寄存器正代表一个虚拟中断，不可用。

`ich_vtr_el2`：[4:0]位等于List寄存器的数量减一。

## 不重要的知识

### 外设是怎么将中断发给GIC的？

2种方式，一种是发送物理中断信号；一种是写GIC的寄存器产生中断，这又称为MSI（message signaled interrupt）。外设发送中断后，GIC就可以根据外设或MSI寄存器来确定INTID了。

### Edge-triggered和level-sensitive的中断区别是什么？

edge-triggered：只要外设发出中断信号，那么只有当软件承认了该中断，中断才会消失

levle-sensitive：外设发来的中断信号存在时，该中断保持；中断信号消失，该中断也会消失。

## 问题

- [ ] ITS学习
- [x] 中断处理流程
- [ ] 中断虚拟化不包括虚拟distributor和redistributor，这是软件干的事情。sysHyper怎么干的？