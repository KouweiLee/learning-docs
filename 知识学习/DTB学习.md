# DTB学习

## dtb的结构

DTB分有两种文件, .dtsi和.dts .由于一个SOC可以做成多个板子, 这些板子都会有共同信息, 这些共同信息组合起来就是.dtsi, 其他的.dts直接引用.dtsi文件即可. 

* 节点

每个设备都是一个节点. 节点命名规则如下:

```
node-name@unit-address
```

其中address一般是设备的首地址. 如果这个节点没有地址, 那可以不要. 

也有的节点命名会是这样:`label:节点名`. 使用`&label`就可以方便地访问节点了. 这种方法通常在向设备树节点添加或修改内容时使用, 比如一个SOC的dtsi有一个节点, 当前的板子希望修改这个节点, 但是不能直接修改dtsi, 否则会影响其他板子. 因此在引用dtsi的设备树文件中直接通过`&label {xxx}`就可以修改这个节点了. 

* 根节点

每个设备树文件都有一个根节点, 引用了别的设备树的文件, 这些设备树文件的根节点会合并成一个.

* 属性

描述节点的信息, 是键值对. 

## 属性

### compatible

又称兼容性属性, Linux会根据compatible属性(是一个字符串列表)寻找可用的驱动. 

### #address-cells和#size-cells

这两个属性用来描述一个节点的所有字节点的地址信息, 前者表示reg属性中地址信息的字长, 后者表示reg属性中地址长度所占用的字长(32位). reg属性的值一般是(address, length)对的形式, 表示某个外设的寄存器地址范围. 



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