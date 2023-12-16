# jailhouse学习

## 基本情况

jailhouse又名监狱，cell又名监狱中一个个小的房间。

jailhouse is loaded and configured by a normal Linux system. So you boot Linux first, then you enable Jailhouse and finally you split off parts of the system's resources and assign them to additional cells.

Jailhouse启动部分：[jailhouse/Documentation/bootstrap-interface.txt at master · siemens/jailhouse (github.com)](https://github.com/siemens/jailhouse/blob/master/Documentation/bootstrap-interface.txt)

启动后，root cell如果启动新的cell，则会让出一些资源的控制权：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309162149281.png" alt="在这里插入图片描述" style="zoom: 50%;" />

因此jailhouse非常简单，没有调度一说，不同的cell占用不同的CPU核心和硬件资源。

* 有关cell的描述

cell就是整个系统被分成的独立单位，每个cell运行一个guest并有对应的CPU、内存、PCI设备。

* **jailhouse编译完成后各个文件夹的说明**

configs/arm64里的xxx.cell文件是vm配置文件，在enable和cell create时使用

driver/jailhouse.ko 是加载到root cell的内核模块，提供部分管理功能，通过ioctl操作为用户态提供接口，使用hypercall调用hypervisor提供的接口。

hypervisor/jailhouse.bin 为jailhouse的hypervisor镜像，运行在EL2，提供虚拟化的核心功能，本项目的sysHyper主要就是实现对它的替换。

tools/jailhouse是root cell的用户态程序，用来调用内核模块jailhouse.ko，是面向用户的管理接口。

inmates 里面是一些可运行的demo。

因此，jailhouse分为三部分：

el0: tools/jailhouse

el1: driver/jailhouse.ko

el2: hypervisor/jailhouse.bin

* **struct jailhouse_system结构体**

用于描述系统。包含3个部分：

* hypervisor_memory, which defines Jailhouse's location in memory;

* config_memory, which points to the region where hardware configuration is stored (in x86, it's the ACPI tables); and

* system, a cell descriptor which sets the initial configuration for a Linux cell.(jailhouse_cell_desc)

紧随jailhouse_system之后，有几个部分：

* cpu_set: a bitmap which lists cell's CPUs (1 at Nth position means Nth CPU is assigned to the cell);

* mem_regions: an array which stores physical address, guest physical address (virt_start), size and access flags for this cell's memory regions. There can be many of them: RAM (currently it must be the first region), PCI, ACPI or I/O APIC. See config/qemu-vm.c for an example;

* irq_lines: an array which describes IRQ lines. It's unused now and may disappear or change in future;

* I/O bitmap (usually referred to as pio_bitmap), which controls I/O ports accessible from the cell (1 means the port is inaccessible). This is x86-only, since no other conceivable architecture has separate I/O space;

* pci_devices: an array, which maps PCI devices to VT-d domains.

### 内存布局

Jailhouse的内存是一段物理上连续的区域，虚实地址转换只需要一个偏移量。总的物理内存布局图如下：

```c
    +-----------+-------------------+-----------------+------------+--------------+------+
    | Guest RAM | hypervisor_header | Hypervisor Code | cpu_data[] | system_confg | Heap |
    +-----------+-------------------+-----------------+------------+--------------+------+
                |__start                              |__page_pool                	     |hypervisor
_header.size
```

jailhouse管理内存通过一个基于bitmap的页式分配器，page_init函数初始化内存管理。

system_config就是cell文件中的jailhouse_system，启动时会将其拷贝过去

system_config后面实际上是mem_pool的bitmap。guest ram就是guest代码段等所在的位置。

详见[memory layout](memory-layout.txt)

## 启动jailhouse

* 编译Jailhouse：

在jailhouse文件夹下，执行

```
make ARCH=arm64 CROSS_COMPILE=~/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=~/study/hypervisor/linux
```

* 启动Linux

在host文件夹中执行：

```
qemu-system-aarch64 \
-drive file=./rootfs.qcow2,discard=unmap,if=none,id=disk,format=qcow2 \
-m 1G -serial mon:stdio -netdev user,id=net,hostfwd=tcp::23333-:22 \
-kernel jail-img \
-append "root=/dev/vda mem=768M"  \
-cpu cortex-a57 -smp 16 -nographic -machine virt,gic-version=3,virtualization=on \
-device virtio-serial-device -device virtconsole,chardev=con -chardev vc,id=con -device virtio-blk-device,drive=disk \
-device virtio-net-device,netdev=net
```

> jail-img是跑jailhouse的linux内核

进入qemu后，执行：

```
cd jail
insmod ./jailhouse.ko
cp jailhouse.bin /lib/firmware/
./jailhouse enable configs/qemu-arm64.cell
```

> insmod，动态将jailhouse.ko内核模块载入linux内核中。通常使用insmod的内核模块是驱动程序。

如果要关闭jailhouse，执行：

```
jailhouse disable
```

* 新建guest cell

```c
jailhouse cell create /path/to/qemu-arm64-inmate-demo.cell
jailhouse cell load inmate-demo /path/to/gic-demo.bin  // 其中inmate-demo是cell的name
jailhouse cell start inmate-demo
// 杀死 cell
jailhouse cell destroy inmate-demo
sudo ./tools/jailhouse cell destroy gic-demo

```

.cell 文件见对应的C文件

### 启动一个non root linux

命令：

```shell
sudo ./tools/jailhouse cell linux ./configs/arm64/qemu-arm64-linux-demo-backup.cell ~/jailhouse/Image -c "console=ttyAMA0,115200 root=/dev/ram0 rdinit=/linuxrc" -d ./configs/arm64/dts/inmate-qemu-arm64-backup.dtb -i ~/guest/initramfs.cpio.gz
```

root cell内核态会通过`execvp`执行tools/jailhouse-cell-linux的python脚本，该脚本会将linux-loader.bin、kernel image、initrd load进去，并将kernel和dtb的地址信息load到.cmdline中，脚本最后执行cell.start时，会跳入cpu_reset_address也就是0x0执行__reset_entry函数，最终会执行inmate_main函数，即linux-loader程序，该程序会disable掉MMU，以便kernel的启动，并将dtb的地址作为参数跳转到Kernel的text地址进行执行。

> objcopy从.o到.bin时，不会复制开头部分的空白无效部分。

关闭这个non root linux：

```
sudo ./tools/jailhouse cell destroy qemu-arm64-linux-demo
```



* 参考资料

[Arm64下Linux内核Image头的格式_arm64 kernel image header-CSDN博客](https://blog.csdn.net/Roland_Sun/article/details/105144372)

### 双串口启动

```
sudo qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 16 \
    -m 1024 \
    -kernel Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=/home/lgw/study/hypervisor/ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -net nic \
    -net user,hostfwd=tcp::2333-:22 \
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3 
```

make run后，在新终端执行sudo screen /dev/pts/5，然后回车。这样root linux用的就是virtio-serial，non root用的物理串口。

## debug jailhouse

host文件夹下执行脚本debug_jail.sh启动qemu。

gdb：

```
gdb-multiarch
target remote:1234
set arch aarch64
c
```

linux:

```
sudo insmod driver/jailhouse.ko
cd /sys/module/jailhouse/sections
sudo cat .text
sudo cat .data
sudo cat .bss
```

gdb:

```
add-symbol-file ../real_jailhouse/drivers/jailhouse.ko 0xffffd6dbae0fd000 -s .data 0xffffd6dbae104000 -s .bss 0xffffd6dbae104900
b enter_hypervisor
```

linux:

```
./jailhouse enable configs/qemu-arm64.cell
```

在gdb中，位置于 <enter_hypervisor+68>，跳入arch_entry函数。

el2_entry的地址为：0x7fc04070

jailhouse：0xffffc0200000

bootstrap_vectors:

差值：0xffff0000bfa00000

## 2种文件

.cell：cell的配置文件

.bin：有一定内存布局的可执行文件，包含代码段等等，对于jailhouse.bin则是以header开头

## enable jailhouse

在root Linux下执行`./jailhouse enable configs/qemu-arm64.cell`命令时，用户态程序tools/jailhouse.c根据enable参数，会执行enable函数

### tools-el0

jailhouse.c

1. enable：通过ioctl发送JAILHOUSE_ENABLE，jailhouse.ko会捕捉到该系统调用，跳转到jailhouse_cmd_enable

> ioctl 是设备驱动程序中设备控制函数（一种系统调用），用户态程序可通过ioctl向特定的设备发出命令和参数。这里将jailhouse.ko当成了一种设备驱动

### driver-el1

linux loader driver

main.c：jailhouse_ioctl->

1. jailhouse_cmd_enable

   该函数通过调用request_firmware函数，将jailhouse.bin载入内存，并进行了物理地址和虚拟地址的映射，形成了指定布局的Hypervisor(commonly visible hypervisor memory)

   通过jailhouse_cell_prepare_root(&config->root_cell)，为该cell注册了sysfs上的一个kobject。

   每个CPU调用enter_hypervisor。

2. enter_hypervisor：该函数内调用header中的arch_entry，并将CPU id作为参数传入

### hypervisor-el2

arch/arm64/entry.S

1. arch_entry：

清空了从hypervisor core中的D-cache，以防当执行hvc时，MMU enable之前，数据还会被保存在D-cache中。

将异常向量表bootstrap_vectors保存在x1中，x1是真实的物理地址。bootstrap_vectors中的el2_entry就是执行hvc时跳转到的异常处理入口。

在el2_entry中，由于没有开启MMU还，PC指向的都是真实物理地址。在init_bootstrap_pt中初始化了bootstrap页表，具体来讲JAILHOUSE_BASE和trampoline进行了地址映射，接着执行enable_mmu_el2函数，设置bootstrap_pt_l0为一级页表，该函数代码存放在trampoline页面，trampoline页面比较特殊，其虚拟地址与物理地址相等，以使得开启mmu后PC能平滑过渡。

之后设置了vbar_el2为hyp_vectors，当异常进入el2时，会跳入该异常向量表。又初始化了进入entry的参数，其中percpu就对应于hypervisor内存布局中的percpu，并将x19-x30寄存器保存在了percpu中，其中x30为x17，即返回el1的地址。

跳转到下面的entry函数

#### setup.c（重点）

每个CPU调用，**初始化Jailhouse和root cell**

1. entry：

master CPU调用init_early：

* paging_init：创建2个page pool，mem_pool用来表示从`__page_poll`到hypervisor区最顶端区域内的所有pages，remap_pool还不太清楚。

  初始化了hv_paging_structures，并将root Linux映射的hypervisor区域，映射到了hv_paging_structures.root_table当中。（TODO：sysHyper中暂时未找到）
  
* mmio_cell_init：为mmio控制区域分配内存即mmio_region_location[n]和mmio_region_handler[n]

每个CPU调用cpu_init：

* arch_cpu_init：使能MMU，将cpu_data中的页表基地址存入TTBR0_EL2，作为MMU的页表基地址；写hcr寄存器，开启虚拟化，效果为：
  * enables stage 2 address translation for the EL1&0 translation regime
  * Physical IRQ interrupts are taken to EL2
  *  If EL3 is implemented, then any attempt to execute an SMC instruction at EL1 is trapped to EL2
  * EL1 accesses to the specified registers are trapped to EL2
  * When EL1is capable of using aarch32, the Execution state for EL1 is AArch64. 
* arm_cpu_init：设置了VTTBR_EL2寄存器，保存el1&0中stage 2地址转换的page table基地址（真实物理地址），以及对应的VMID；设置了VTCR_EL2寄存器，用于控制stage 2的地址转换。最后执行irqchip_cpu_init初始化gic

init_late：

* 初始化units，调用pci和irqchip的init函数
  * irqchip_init：注册gicr和gicd的mmio区域的访问，以及handler函数。（对于non root cell，会将GIC的bitmap和root cell的bitmap做隔离）
  * pci_init：
  
* **为root cell映射填充stage 2页表**

#### arch/arm64/setup.c（重点）

1. arch_cpu_activate_vmm

取消CPU的private data在hv_paging_structs页表中的映射。

调用arch/arm64/entry.S中的vmreturn，将cpu data中的guest regs恢复到寄存器堆，最终执行eret返回el1（返回地址就是hvc的下一条指令）

> vmreturn也在跳板页面中，保证平滑返回

内存布局可见hypervisor.lds.S

## 页表映射

共有3种页表：

### hypervisor el2页表

该页表不会包含per_cpu中的private区域

```c
hv_paging_structs.root_table = (page_table_t)public_per_cpu(0)->root_table_page;
```

> 话说在取消这个页表对所有CPU private域的映射时，不应该取消0号CPU呀，因为这就是

per_cpu中的pg_structs：每个per_cpu都在public域中保存着一张根页表，根页表指向的下一级页表其实和hv_paging_structs中的root_table一样了。对于JAILHOUSE_BASE这个有自己per_cpu中的private域映射。**该页表会作为本CPU在el2时的页表基地址**（设置到MMU里了）

* el2地址空间包含的内容：

hv_paging_structs：

hypervisor core：在paging_init函数中将linux映射的hypervisor core映射给了el2的地址空间

percpu private：paging_map_all_per_cpu函数中

debug_console_base：paging_init函数中

trampoline_page: 在arch_init_early中

gicd/gicr：在irqchip_cpu_init中

cpu 本身的el2页表：

继承了JAILHOUSE_BASE、debug_console_base的子页表(包含了gicd这些应该都)

cpu data 的private mapping

### cell的stage2页表

cell的stage 2地址转换页表：*cell*->arch.mm.root_table，在arch_map_memory_region函数中为其将cell->config中的各个jailhouse_memory区域指定的virt(IPA)和phys(PA)进行了映射。

**对于root_cell：stage 2为恒等映射**

对于非root_cell：其ram内存区域的映射，IPA从0开始，映射到对应的PA；其映射是通过cell配置文件来的。对于subpage region，将注册为mmio区域，不添加到stage 2页表，届时会由

### cell内核态的stage 1地址转换

由本cell自己管理，hypervisor并不管。从该cell的角度，该页表就是映射到真实的物理地址。

## GIC相关

jailhouse中有关GIC的配置：

```c
.arm = {
    .gic_version = 3,
    .gicd_base = 0x08000000, // gicd所在的物理地址
    .gicr_base = 0x080a0000, 
    .maintenance_irq = 25,
},
```

在enable jailhouse时，所有CPU会执行arm_cpu_init函数（不过是互斥的执行），最终会调用irqchip_cpu_init，初始化GIC

```c
irqchip_cpu_init(struct per_cpu *cpu_data)
```

master CPU会为irqchip赋值为gicv3_irqchip，并将gicd、gicr分别映射到el2的虚拟地址上，该虚拟地址保存在gicd_base和gicr_base。由于是修改hv_paging_structs的映射关系，故所有CPU都可见。

gicv3的gicd_base的值为：0x10000

```
#define GIC_V3_REDIST_SIZE	0x20000
cpu数量*GIC_V3_REDIST_SIZE 就等于 gicr的size
```



* 初始化GIC：

gicv3_cpu_init：

首先寻找该CPU对应的redistributor，通过GICR_TYPER寄存器寻找。

设置GICR_ISENABLER寄存器：开启SGI和指定的mnt_irq ppi到CPU interface的转发。

ICC_PMR_EL1：设置priority最低为0xf0，否则不会转发给PE。priority数值越低越优先

ICC_IGRPEN1_EL1：*enable Group 1 interrupts*，允许对OS和HYpervisor的中断

清空pending irqs

开启vGIC：ICH_VMCR_EL2，设置Virtual Priority Mask（中断优先级的最低要求），开启虚拟Group 1中断，开启写End of Interrupt寄存器时既drop priority又终止中断；ICH_HCR_EL2，开启virtual CPU interface

### SGI

首先构造sgi：

```c
struct sgi {
	/*
	 * Routing mode values:
	 * 0: use aff3.aff2.aff1.targets
	 * 1: all processors in the cell except this CPU
	 * 2: only this CPU
	 */
	u8	routing_mode;
	/* cluster_id: mpidr & MPIDR_CLUSTERID_MASK */
	u64	cluster_id;
	u16	targets;
	u16	id;
};
```

通过以下两个函数来构造targets 和 cluster_id。

```c
static int gicv3_get_cpu_target(unsigned int cpu_id)
{
    // mpidr & 0xff表示寄存器的编号，将1左移若干个编号则对应targetlist
	return 1 << (public_per_cpu(cpu_id)->mpidr & MPIDR_AFF0_MASK); // 0x000000ff, aff0
}

static u64 gicv3_get_cluster_target(unsigned int cpu_id)
{
	return public_per_cpu(cpu_id)->mpidr & MPIDR_CLUSTERID_MASK; // 0x00ffff00, aff2.1
}

```

之后在gicv3_send_sgi函数中写ICC_SGI1R_EL1寄存器，发出sgi。该寄存器中TargetList，其中第n位为1，表示发给第n个CPU。

收到中断的PE，会根据异常向量表依次执行：arch_handle_exit---->irqchip_handle_irq---->arch_handle_sgi--(event)-->check_events。

对于suspend_cpu来说，在check_events中会执行cpu_relax直到suspend_cpu被取消。

对于park cpu来说，会设置ELR返回地址为0，以及对应的stage 2页表为parking_pt。parking_pt的ipa为0的地址就是wfi指令，只有当会trap到el2的中断发生时，才会取消wfi的状态。据说reset cpu时会将stage 1禁用。

对于reset cpu来说，会指定cpu异常返回时的pc值，这样就可以执行cell的bin文件了

## mmio

guest访问mmio区域时的调用流程：

arch_handle_dabt->mmio_handle_access->对应mmio region的handler函数。在handler函数中，mmio->address是相对于所在region的偏移量。

### handler函数总结

对于gicr所在区域，handler为：gicv3_handle_redist_access，其arg为public_per_cpu(cpu)

对于gicd所在区域，handler为：gic_handle_dist_access；如果handler中几个分支都未命中，则执行gicv3_handle_dist_access。

对于subpage的mmio区域，handler为mmio_handle_subpage

### pci设备

在pci.c中，通过`DEFINE_UNIT(pci, "PCI");`定义了一个`pci_unit`。

在enable hypervisor时，init_late函数中，会调用pci_init.

在cell_create时，会对每个unit调用cell_init，即pci_cell_init函数。

#### pci设备简介

PCIe目前已大规模取代pci，可以将它们当成一样的。

bdf唯一标识pci设备的一个功能，即bus, device, function，分别占8,5,3位。即某根总线上的某个设备的某个功能。

<img src="https://img2018.cnblogs.com/i-beta/1813174/201911/1813174-20191128162041950-420448968.png" alt="img" style="zoom: 67%;" />

其中root complex是CPU和PCIe总线系统通信的媒介。endpoint则是一个个pci设备，作为总线操作的发起者和终结者。

* PCIe的配置空间

每个PCIE设备都有自己的独立的一段配置空间。配置空间中，每个function有256B的配置空间，前64字节是Header Space，有两种类型，Type0和Type1，因为PCIe设备分为Bridge和endpoint两种类型，endpoint的配置空间类型称为Type 0类型，Bridge的配置空间类型称为Type1类型。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/20210731114759614.png" alt="img" style="zoom:67%;" />

这两种类型的设备在PCI拓扑结构中的位置如下：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/20210807104039137.png" alt="img" style="zoom:67%;" />

* 配置空间中的BAR寄存器

在配置空间的header中，有一些bar寄存器，即base address registers，用来指示该设备的配置空间基地址。由于配置空间可分为MMIO或IO类型，根据bar bit 0的值可确定类型：

<img src="https://img-blog.csdnimg.cn/20210807120652771.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTMwNzU=,size_16,color_FFFFFF,t_70" alt="img" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210807120847842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTMwNzU=,size_16,color_FFFFFF,t_70" alt="img" style="zoom:50%;" />

#### ivshmem

虚拟机间共享内存设备ivshmem (Inter-VM shared memory device)，用于guest之间共享内存。QEMU将该设备建模为一个PCI设备，将共享内存作为PCI的BAR空间。

* 中断机制——MSI-X

MSI，message signal interrupt, 是PCI设备通过写一个特定消息到特定地址，从而触发一个CPU中断。

## 问题

- [x] 什么是driver：el1下的jailhouse.ko
- [x] **.cell文件是怎么产生的：**在config中对应c文件编译过来的
- [x] system配置文件的内存布局是什么？它从哪里来

应该是从cell文件来，cell文件是从对应的C文件来。

- [x] 特权级切换的时候，页表会自动切换吗？页表基地址在MMU中被保存在TTBR_EL2.（这个感觉得看看entry.S的汇编代码了）

如果到达的特权级指定了页表基地址并enable 了mmu，则会自动切换

- [x] enable el2 mmu之前，用的是哪个mmu？

执行过2次在初始化jailhosue的时候，不执行则不使用mmu

- [x] per cpu这个作用是什么

应该就是描述这个CPU的一个数据结构，其中包括cpu id、所属cell、paging structures（用于页表等相关操作）

- [x] 多核CPU，是不是所有寄存器包括系统寄存器、特殊寄存器、TLB每个核都有一套？

是的。可以总结一下，多核CPU

- [x] root cell传进来的指针参数，如何与hypervisor el2的地址统一？传的时候是guest 物理地址，进行转换即可。
- [x] stage 2页表映射的内容是什么？

jailhouse_memory中规定的那些

- [x] jailhouse memory中的 各个域是什么意思：*phys_start表示真实物理地址，virt_start表示IPA*
- [x] hypervisor自己的页表是怎么映射的。

这里要考虑多核概念。多核，每个核MMU是不同的，页表基地址是自己 per_cpu中的root_table。

在使能el2 MMU和开启虚拟化之前，el1的PA是真实的PA而不是IPA。刚进入el2时，JAILHOUSE和跳板页面这些虚拟地址会被映射到对应的物理地址上。之后，在paging_init会将hypervisor整个区域映射到hv_paging_structs中，虚拟地址从Jailhouse开始，虚实地址转换其实只需要一个偏移量。每个CPU会根据此来构建自己的映射页表，并将其作为自己el2页表。

- [x] cell的stage 1页表在哪里设置？映射关系是什么？

TTBRx_EL1保存了stage 1地址转换的页表基地址。这个不归hypervisor管，root cell以前就是有个偏移量。stage 2映射是恒等映射。

- [x] 话说root cell的stage 2地址转换，是1:1的吗？

是，恒等映射，见qemu-arm64.c文件

- [x] cell结构体中的这个是什么玩意

```c
	union {
		/** Communication region. */
		struct jailhouse_comm_region comm_region;
		/** Padding to full page size. */
		u8 padding[PAGE_SIZE];
	} __attribute__((aligned(PAGE_SIZE))) comm_page;
```

与hypervisor通信的区域应该是

- [x] per cpu是怎么影响真实的cpu的？

cpu如果要suspend，那么可能就是通过per cpu中的字段维持在一个循环里，这样另一个cpu修改这个字段，就可以“唤醒”它。也就是说，per cpu就像一个共享内存，不同cpu可以进行通信。

- [x] cell启动后，又是怎么知道启动地址进行启动的呢？

当PE进入sgi异常处理函数后，最后会执行`arm_cpu_reset(*cpu_public*->cpu_on_entry)`，这样从EL2返回时，就会在cpu_on_entry处执行了

- [x] unit是什么

unit通过宏DEFINE_UNIT定义，并被放在.units代表的地址上。目前只有irqchip和pci。

- [ ] GzipFile是什么？启动arm64 non-root-linux时，是否需要将Image转换成Gzipfile
- [x] 这里的.cmdline是什么时候被链接的？

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310121822255.png" alt="image-20231012182236040" style="zoom:50%;" />

通过`readelf -S linux-loader-linked.o`命令，在其中找到了.cmdline段。在jailhouse-cell-linux中，在non-root-linux start之前，会调用函数：

```python
# 向.cmdline中增加kernel和dtb的地址信息
cell.load(arch.params, arch.params_address())
```

这就在.cmdline中增加了kernel和dtb的地址信息

- [x] 为什么启动一个cell要分成create、load和start 3 步？

create的主要任务：

创建新cell的控制结构体，并将新cell 的cpu从root cell中脱机，并置为park状态。

将新cell的mem区域从root cell的stage 2页表解除映射，之后root cell就不能访问新cell的所有mem表示的物理地址区域了。

将新cell的mem区域映射到新cell的stage 2页表.

load的主要任务：

将新cell要加载镜像的mem区域映射回root cell的stage 2页表，以便其可以将镜像加载到对应的物理地址。

之后root cell将新cell的镜像加载到指定的mem物理地址区域

start的主要任务：

首先将load时映射到root cell的新cell区域的mem解除映射

将新cell的第一个CPU置为cpu_reset_address，其他cpu设置为park状态（Wfi）。

- [ ] SGI_INJECT和SGI_EVENT是什么？

## 参考资料

### pci

bar: https://blog.csdn.net/u013253075/article/details/119361574

