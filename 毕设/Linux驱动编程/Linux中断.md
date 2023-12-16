# Interrupts in Linux Kernel

## Linux中断需要注意的地方

1. 中断处理函数不能发生sleep或者阻塞，因此不能使用Mutex。如果想加锁，使用spinlocks
2. 中断处理函数不能和用户空间交换数据
3. 

## 参考资料

1. https://embetronicx.com/tutorials/linux/device-drivers/interrupts-in-linux-kernel/
2. https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/