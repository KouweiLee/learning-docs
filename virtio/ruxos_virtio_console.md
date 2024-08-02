# ruxos virtio console实现文档

for_each_drivers里要加入virtio_console。

rust_main--》

init_drivers--》

AllDevices::probe-->这会将设备加入到AllDevices的container里面。

probe_bus_devices进行mmio区域的探测-->

​	probe_mmio-->

​		probe_mmio_device：获得MmioTransport

## console

步骤：

1. 做好适配
2. 中断处理函数加上

## blk

blk的实现，重点函数是read_block和write_block，这两个函数都是通过轮询等待读写操作完成，而不是中断。

## TODO

- [x] ruxos的到达virtio的启动过程
- [ ] ruxos virtio如何调用第三方库

