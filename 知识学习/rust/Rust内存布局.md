# Rust内存布局

一个类型的布局layout，包含3个组成部分：

1. alignment：如果一个类型的alignment为n，则该类型实例的起始地址能整除n

2. size：将类型看作数组中的一个元素，那么两个相邻元素之间一个元素所占空间加上padding的大小就是size。可以推断出，size必须是alignment的倍数。

如果一个类型的所有实例，都有相同的size和alignment，且在编译期就可以获知，则这个类型自动实现了`Sized`特征，并可以通过`size_of`和`align_of`获取（单位字节）。否则，该类型称为动态大小类型。

## 结构体的内存布局

每个结构体类型，都有一个representation，这个representation决定了它的layout。一个结构体类型的representation可以是：

1. 默认的
2. C
3. 还有些不常见的

通过`#[repr]`可以改变结构体的representation。

### 默认representation

如果不指定结构体的representation，则默认为Rust representation。这样的layout，只会保证满足各个filed的alignment，以及fileds之间不互相重叠，但**不保证各个filed的顺序与结构体代码的一致**。

结构体类型的alignment是所有fileds中最大的alignment。

### C representation

如果通过`#[repr(C)]`指定了representation，那么就与C语言的layout相同了。各个filed之间的顺序和结构体代码相一致，且满足alignment要求，整个类型的alignment为所有fileds中最大的alignment。

注意，representation不会改变filed的layout，也就是说如果结构体之间出现嵌套，那么改最顶层的结构体的representation不会影响底下的结构体。

### alignment修饰符——align&packed

使用`#[repr(align(n))]`，可以增加结构体类型的alignment，如果n小于结构体类型本身的alignment，则没用。

使用packed，可以减小结构体类型的alignment。结构体中所有fileds的alignment，是指定的和本身的两者中的最小值，且fileds之间的padding保证是满足alignment的最小值。`#[repr(packed(1))]`和`#[repr(packed)]`会使得fileds之间没有padding

## 参考资料

官方：https://doc.rust-lang.org/reference/type-layout.html#type-layout