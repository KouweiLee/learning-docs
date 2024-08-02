# 开题报告-一种虚拟机监控器中VirtIO后端机制设计与实现

## 选题背景

虚拟机监控器（Hypervisor），是一种创建并运行虚拟机(VM)的软件，可用于在单个计算机上运行并管理多个虚拟机。其中每个虚拟机都有自己的操作系统和应用程序。虚拟机监控器会根据需要将 CPU、内存、硬盘和网卡等硬件资源虚拟化，分配给各个虚拟机。多个虚拟机之间在Hypervisor的协调下可以共享这些硬件资源，同时每个虚拟机也可以保持各自的独立性。

虚拟化技术中一个重要的研究领域是设备虚拟化。设备虚拟化是指用软件的方式模拟操作系统的各种外设，比如磁盘、网卡、GPU等。Hypervisor会将这些虚拟设备提供给虚拟机，虚拟机只要实现了对应的驱动程序即可使用这些虚拟设备。在设备虚拟化的发展历程中，先后有多种虚拟结构被提出，分别为：全虚拟化、半虚拟化和硬件辅助虚拟化。

全虚拟化：全虚拟化是指Hypervisor遵循硬件的规范，完整模拟硬件逻辑，拦截虚拟机对该设备的所有访问操作，并且用代码模拟设备的功能。这种方式对虚拟机操作系统是透明的，操作系统不需要做任何修改即可使用虚拟设备。但全虚拟化实现起来较为复杂，且由于其完全遵循硬件规范，性能较低。

半虚拟化：半虚拟化是指虚拟设备与虚拟机操作系统驱动程序之间按照约定好的接口进行通信，相比全虚拟化完全遵循真实设备的访问方式，这种交互方式更加简洁高效。Virtio 是一种典型的半虚拟化实现方案，虚拟机的驱动程序和Hypervisor的虚拟设备之间的通信通过共享内存实现，不再使用寄存器等传统的 I/O 方式，降低了设备模拟的复杂度，减少了CPU 在 guest 与 host 之间的切换，大大提高了虚拟机的 I/O 性能。

硬件辅助虚拟化：硬件辅助虚拟化是一种需要借助硬件来模拟设备的技术，比如：Intel VT-d 支持将整个设备透传给虚拟机，这样虚拟机就可以直接访问硬件设备，而不需要通过虚拟化软件来访问硬件设备，但这种方式无法在虚拟机之间共享设备, 设备利用率低.

在这3种虚拟化技术中，本文聚焦于研究和实现半虚拟化技术中的Virtio方案。Virtio协议于2008年提出，2016年正式发布1.0版本,是一种旨在提高设备性能, 统一各种设备模拟方案的设备虚拟化标准. 它通过定义通用的设备规范和协议，为虚拟机提供了高性能、通用化的虚拟设备访问方式。Virtio协议囊括了多种外设如磁盘, 网络, 串口等, 规定了驱动程序(前端)和虚拟设备(后端)之间通信的接口. 其中控制面接口采用PCI或MMIO的方式, 数据面接口采用共享内存Virtqueue进行数据传输. 

- [x] 如图, 写一个virtio driver和device的结构图

![image-20240222111232992](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240222111232992.png)

目前许多操作系统包括Linux均实现了Virtio驱动程序, 因此为hypervisor实现Virtio设备对于提高设备利用率是十分重要的.

## 研究现状

virtio半虚拟化技术的应用已经相当广泛, 已经有许多虚拟机监控器根据自身需求按照Virtio协议实现了各式各样的Virtio设备. 

### QEMU

QEMU是一款由Fabrice Bellard等人编写的硬件仿真器和模拟器, 本质上是一个Linux用户程序, 配合KVM(Kernel-Based Virtual Machine, Linux中的一个内核模块)可高效地运行多种虚拟机. KVM为QEMU提供了一系列接口用来创建, 配置和启动虚拟机. 当虚拟机操作系统中Virtio驱动程序发出IO请求时, 会陷入到KVM, KVM会将这些请求提交给QEMU进行处理. 作为一个Linux进程, QEMU则最终会将这些请求通过POSIX标准接口对应到文件, 网络设备等物理资源进行处理. 处理完成后再通过中断注入通知客户机操作系统. 上述流程如下图所示:

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240221183415187.png" alt="image-20240221183415187" style="zoom: 80%;" />

QEMU实现的Virtio设备种类最为齐全, 包括Virtio blk, net, gpu等多种虚拟设备. 

### ACRN

ACRN是一个由英特尔使用C语言开发的轻量级Hypervisor, 面向物联网和嵌入式场景, 支持x86体系结构, 可直接运行在真实硬件上, 并启动多个虚拟机. 其中称为Service VM的客户机包含一个Device Model进程,  该进程提供Virtio设备的后端实现. 当其他VM(称为User VM)发出IO请求时, 该请求会依次经由ACRN和HSM(一个Linux内核模块)发送给Device Model, 当Device Model完成IO模拟时, 会将结果依次通过HSM和ACRN返回给User VM. 如下图所示, 整个IO请求的传递路径较长, 导致Virtio设备性能较差. 

![virtio-hld-image3](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/virtio-hld-image3.png)

### Rust-Shyper

Rust-Shyper是由北京航空航天大学使用Rust语言开发的面向嵌入式环境的hypervisor, 可直接运行在物理硬件上. 它实现了3种Virtio设备, 分别为Virtio-blk, Virtio-net和Virtio-Console. 这3种设备的数据面和控制面接口均位于hypervisor层, 但是通过利用Linux内核设备驱动代替Hypervisor对真实外设进行操作, 简化了hypervisor设计和实现的复杂度. 例如对于Virtio-blk设备来说, 通过利用MVM(管理虚拟机) 对磁盘上的一个镜像文件进行读写来模拟磁盘设备的数据传输. 

## 研究内容

### 研究基础

本研究的基础是矽望开源社区正在开发的虚拟机监控器hvisor。hvisor是一个使用高级语言Rust实现的轻量级Type-1型虚拟机监控器，基于ARMv8体系结构，面向嵌入式场景，主要为虚拟机提供硬件资源虚拟化、分区安全隔离的功能。hvisor启动时会首先启动root linux虚拟机，该虚拟机包含一个单独的内核模块，可以发起Hypercall与hvisor提供的私有特权接口通信，从而监控和管理其他虚拟机。目前hvisor的设备虚拟化实现方式仅支持设备直通，虽然设备直通实现简单、性能最好，但设备利用率低，无法给虚拟机提供不存在的设备。因此目前hvisor需要实现一套virtio设备，实现设备的半虚拟化，提高设备的利用率。

### 研究目标

在轻量级Type-1型虚拟机监控器hvisor的基础上，实现一套virtio设备，包括磁盘设备、网络设备、GPU等，使其可以为虚拟机提供各种虚拟设备。然而由于hvisor的轻量级特性的限制，virtio设备不能实现在hypervisor层，否则hypervisor层的代码量将过于庞大。因此本文目标希望在root linux的用户态实现virtio设备，当其他虚拟机virtio驱动程序要与virtio设备进行通信时，hypervisor作为一个跳板将请求转发给root linux的用户态virtio进程，当virtio进程完成该请求对应的操作时，再通过hypervisor将结果返回给其他虚拟机。这样，经过一定的优化手段，既可以保证hypervisor层的轻量性，又可以提供性能较好的虚拟设备。

### 研究内容

hvisor的virtio设备实现分为virtio设备进程, 内核模块, 通信跳板三个模块. 具体设计图如下:

![image-20240221151957563](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240221151957563.png)

#### virtio设备进程

virtio设备进程包含了virtio设备实现的主体代码, 是一个位于root linux用户态的进程. 类似于qemu进程, 它为其他虚拟机的virtio驱动程序提供virtio设备模拟. 本项目计划在virtio设备进程中实现3种设备, 分别为virtio-blk, virtio-net和virtio-Console. 当该进程收到其他虚拟机产生的IO请求时, 会根据这些请求对应的virtio设备进行分发, 交由对应的设备后端进行处理. 例如virtio-blk IO请求则交由virtio-blk设备进行处理, virtio-net IO请求则交由virtio-net设备. 当设备完成IO请求后, 则会通知内核模块并返回结果. 收发IO请求和结果的过程将在下文第四部分介绍. 

#### 内核模块

hypervisor服务内核模块(又称为hypervisor service kernel module, 简称HSM), 是virtio设备进程与hypervisor通信的桥梁, 位于root linux的内核态. 在virtio设备初始化时, HSM需要为virtio设备进程和hypervisor之间建立一片共享内存区域, 存放IO请求和IO结果队列. 当virtio设备进程完成IO请求并通知HSM后, HSM会通过hypercall通知hypervisor.

#### 通信跳板 

通信跳板是root linux和其他虚拟机之间传递IO请求和结果的桥梁, 包含2个队列, 分别存放IO请求和结果. virtio设备进程可以访问这两个队列所在的内存区域. 当某个虚拟机发出IO请求时, hypervisor会将该请求加入到IO请求队列中, 这时监视IO请求队列的virtio设备主线程则会取出该请求进行处理. 当完成该请求后, virtio设备进程会将结果放入到IO结果队列中, 并通过Hypercall通知hypervisor, hypervisor随即将结果返回给虚拟机.  

## 拟解决的关键问题

### virtio设备在用户态的实现

本项目计划实现3种virtio设备: virtio-blk, virtio-net和virtio-Console. 这3种设备均遵循virtio规范, 控制面模拟MMIO的形式, 数据面通过VirtQueue共享内存与其他虚拟机的驱动程序进行数据传输. 由于3种设备均在Linux用户态实现, 因此最终都会对应于用户态可见的某个资源. 例如, virtio-blk设备上的数据最终存储在真实磁盘上的一个文件, virtio-net设备则通过tap设备和网桥与真实网卡传递报文, virtio-Console设备对应到一个真实的串口或文件. 如何设计virtio设备的控制面和数据面, 以及通过多线程的手段优化设备性能, 则是本项目要解决的关键问题之一.

- [ ] 如下图所示: 可以参考这个

![image-20240221220036066](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240221220036066.png)

### root linux和其他虚拟机之间的通信

本项目的另一个要解决的关键问题是如何将其他虚拟机访问virtio设备的请求发送给位于root linux的virtio设备进程, 以及如何将virtio设备进程处理的结果返回给对应的虚拟机. 为了简化通信路径的长度, 提高virtio设备性能, 规避acrn-hypervisor较长的通信路径带来的性能损失, 本项目采用2个队列分别存放其他虚拟机产生的IO请求和virtio设备进程返回的IO结果. 两个队列是virtio设备进程与hypervisor的一片共享内存区域, 采用生产者消费者模型, hypervisor作为IO请求队列的生产者和IO结果队列的消费者, virtio设备进程作为IO请求队列的消费者和IO结果队列的生产者, 采用必要的互斥锁和内存屏障保证多核CPU之间数据访问的正确性和一致性. 

## 进度安排

| 周    | 研究安排                                                     |
| ----- | ------------------------------------------------------------ |
| 1-2   | 查阅相关资料, 进行前期准备                                   |
| 3     | 开题答辩                                                     |
| 5-8   | 实现virtio-blk和virtio-net设备的基础版本以及跳板机制, 不考虑性能, 能成功运行. |
| 9     | 中期检查                                                     |
| 10-14 | 对virtio设备进行多线程版本的优化, 并实现virtio-console设备, 并进行性能分析测试 |
| 15-16 | 完成学位论文, 毕业答辩                                       |

## 参考文献

1. virtio: Towards a De-Facto Standard For Virtual I/O Device。 [virtio-paper.pdf (ozlabs.org)](https://www.ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf)

## 一些修改

背景里增加三种虚拟化方式的对比。参考thu毕设

![image-20240222200604043](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202402222006228.png)

设备虚拟化是指用软件的方式模拟操作系统的各种外设，比如磁盘、网卡、GPU等。Hypervisor会将这些虚拟设备提供给虚拟机，虚拟机只要实现了对应的驱动程序即可使用这些虚拟设备。在设备虚拟化的发展历程中，先后有多种虚拟结构被提出，分别为：设备模拟、设备直通和半虚拟化。