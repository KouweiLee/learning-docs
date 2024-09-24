在irq-gic-v3.c中，有代码：

```c
IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);
```

gic_of_init为初始化函数

gic_init_bases

中断处理函数入口：gic_handle_irq