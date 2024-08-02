# riscv学习

## 通用寄存器

共有32个64位寄存器，分别为x0-x31。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20240614103503.jpg" alt="微信图片_20240614103503" style="zoom: 25%;" />

所有特权级共用一个sp寄存器。 

## 特权寄存器

注意，riscv将当前CPU模式保存在软件不可见的硬件中，故意不让代码轻易发现它正在运行的模式。

### M

* mstatus：处理器当前的运行状态

MPP: bits[12:11]，表示陷入M模式之前CPU的处理模式。0：U模式，1：S模式，3：M模式

MPV：bits[39]，表示陷入M模式之前，V的状态。0：处理器运行在非虚拟化模式；1：处理器运行在虚拟化模式。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240614104532345.png" alt="image-20240614104532345" style="zoom: 50%;" />

* 

### HS

* hstatus：处理器当前状态

* hip：和sip一样，指示哪些中断处于等待响应状态

SSIP: 

* hideleg

将中断委托给VS态处理：



* 虚拟化模式下系统寄存器分布：

HS模式下，VMM使用原有S模式的寄存器，以及新增的HS寄存器。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20240614104921.jpg" alt="微信图片_20240614104921" style="zoom: 33%;" />

### S

* sie：使能和关闭S模式下的中断

SSIE：1, 软件中断使能

STIE：5, 时钟中断

SEIE：9，外部中断

* sip：指示哪些中断处于等待响应状态

SSIP：1,软件中断正在等待

STIP：5, 时钟中断

SEIP：9, 外部中断



## 处理器模式

没有虚拟化扩展，有U、S、M三个模式；有虚拟化扩展，则有VU、VS、HS、M四个模式。

## 异常

默认情况下，异常和中断全部在M模式下处理，通过寄存器：medeleg和mideleg，可将部分异常委托给S模式。

* 系统调用

ecall是所有特权级下的特权指令，经过设置相应的委托和代理，ecall可以让CPU陷入不同的异常模式。

* 异常返回

M模式下返回，使用mret；S模式下返回，使用sret。虚拟化下的返回见书P408。

## 问题

- [ ] arm有没有像riscv的异常委托机制。