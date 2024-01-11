# Interrupts in Linux Kernel

## Linux中断需要注意的地方

1. 中断处理函数不能发生sleep或者阻塞，因此不能使用Mutex。如果想加锁，使用spinlocks
2. 中断处理函数不能和用户空间交换数据

## 注册中断

* 注册中断

```c
request_irq
(
unsigned int irq,
irq_handler_t handler,
unsigned long flags,
const char *name,
void *dev_id)
```

参数:

* irq: 要注册中断处理函数对应的中断号
* handler: 中断处理函数
* flags: 定义在**linux/interrupt.h**中的,分别为
  * **`IRQF_SHARED`**: 表示该中断号可由多个设备共享
* name: 使用该IRQ的设备名, 通过cat /proc/interrupts可以看到
* dev_id: 用于识别同一中断号的不同handler

返回值: 成功为0

* 卸载中断

```c
free_irq(
unsigned int irq,
void *dev_id)
```



## 中断处理函数



返回值: 处理成功IRQ_HANDLED, 否则IRQ_NONE

## 参考资料

1. https://embetronicx.com/tutorials/linux/device-drivers/interrupts-in-linux-kernel/
2. https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/