# virtio-i2c

I2C是一种外设总线。I2C只有两条线,一条串行数据线:SDA,一条是时钟线SCL ，使用SCL，SDA这两根信号线就实现了设备之间的数据交互，它方便了工程师的布线。

I2C总线被非常广泛地应用在EEPROM，实时钟，小型LCD等设备与CPU的接口中。

为了方便在linux系统下编写I2C驱动，Linux提供了I2C驱动体系结构，如下图：

![img](https://images0.cnblogs.com/blog/536940/201309/02225054-2c2abb8ed8da431390a03bcbfd6563df.png)



## 参考资料

1. https://www.cnblogs.com/aaronLinux/p/6185882.html