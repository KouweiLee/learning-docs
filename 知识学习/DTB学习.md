# DTB学习

## 属性

### interrupts

interrupts中最后一个值,表示中断是边沿触发还是电平触发, 具体为:

```c
// linux/include/dt-bindings/interrupt-controller/irq.h
#define IRQ_TYPE_NONE		0
#define IRQ_TYPE_EDGE_RISING	1
#define IRQ_TYPE_EDGE_FALLING	2
#define IRQ_TYPE_EDGE_BOTH	(IRQ_TYPE_EDGE_FALLING | IRQ_TYPE_EDGE_RISING)
#define IRQ_TYPE_LEVEL_HIGH	4
#define IRQ_TYPE_LEVEL_LOW	8
```

	vm_service {
		compatible = "hvisor";
		interrupts = <0x00 0x20 0x01>;
	};