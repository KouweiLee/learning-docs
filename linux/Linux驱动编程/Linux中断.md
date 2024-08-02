# Iterrupts in Linux Kernel

## Linux中断需要注意的地方

1. 中断处理函数不能发生sleep或者阻塞，因此不能使用Mutex。如果想加锁，使用spinlocks
2. 中断处理函数不能和用户空间交换数据

## 注册中断

* 注册中断

```c
int request_irq(unsigned int irq, irq_handler_t handler,
                         unsigned long irqflags, const char *devname, void *dev_id)
```

参数:

* irq: 要注册中断处理函数对应的中断号。注意，这个中断号不是中断控制器认为的硬件中断号，而是Linux内核中的软中断号。从硬件中断号转换成软件中断号，需要通过设备树的一些处理函数，详见下文。
* handler: 中断处理函数
* flags: 定义在**linux/interrupt.h**中的,分别为
  * **`IRQF_SHARED`**: 表示该中断号可由多个设备共享
* name: 使用该IRQ的设备名, 通过cat /proc/interrupts可以看到
* dev_id: 用于识别同一中断号的不同handler；如果flags没有`IRQF_SHARED`，那么需要为NULL

返回值: 成功为0；-INVAL表示中断号无效或处理函数指针为NULL，返回-EBUSY表示中断已经被占用且不能共享

* 卸载中断

```c
void free_irq(unsigned int irq, void *dev_id)
```

其中dev_id和注册中断时的要相同。

## 中断处理函数

```c
static irqreturn_t irq_handler(int irq,void *dev_id) {
  return IRQ_HANDLED;
}
```

参数：

* irq：中断号
* dev_id：注册中断处理函数时传入的dev_id。可以通过判断dev_id来区分共享同一个中断号的不同设备。

返回值: 处理成功IRQ_HANDLED, 否则IRQ_NONE

## 从硬件中断号到软件中断号的映射

对于Arm体系结构下，使用设备树来查找设备时，可以通过查找设备树节点来获取软中断号：

```c
// The irq number must be retrieved from dtb node, because it is different from GIC's IRQ number.
struct device_node *node = NULL;
// 查找设备树中节点名为node_name的节点
node = of_find_node_by_path("/node_name"); 
// 查找节点的第0个中断源的软中断号
int irq = of_irq_get(node, 0);
```

这里的irq就是软件中断号了，可以直接用于注册中断处理函数了。

## 参考资料

1. https://embetronicx.com/tutorials/linux/device-drivers/interrupts-in-linux-kernel/
2. https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/