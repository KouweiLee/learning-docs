# riscv hvisor

## 环境配置

QEMU模拟平台：物理内存起始地址0x80000000，bootloader会加载到这个地址，默认是opensbi。-kernel参数指定会从hvisor启动，hvisor所在的物理地址为0x80200000。

host dtb是QEMU传给hvisor的，就是QEMU的模拟平台。另外最开始，只有单核进入Hvisor，由hvisor启动其他核。这时都处于S态。

## hvisor配置

hvisor返回VS态时，设置寄存器：

hstatus：SPV为1, VSXL为2,表示VS态寄存器为64位。

sstatus：SPP为1, 表示要回到VS态

V模式下发生异常陷入HS模式时，进入异常处理函数_hyp_trap_vector。首先保存寄存器到ArchCPU中，之后判断异常类型，同步异常则进入sync_exception_handler，中断则进入interrupts_arch_handle。

## 移植virtio

当客户机发生二阶段地址翻译错误时，会进入guest_page_fault_handler函数

## 问题

- [ ] riscv的inject irq还没写；移植virtio
- [ ] 时不时会卡住，经常性的。。。
- [ ] riscv没有实现关机