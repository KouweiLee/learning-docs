# virtIO学习

virtio分为driver和device，driver部分运行于guest操作系统中，device部分运行于hypervisor中。

基本要素：
<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310141612386.png" alt="image-20231014161248116" style="zoom:33%;" />

## dtb

[一文搞定 Linux 设备树 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/425420889)

目前的方案是，找到qemu中模拟的virt-blk所在的地址，写到linux的dtb里，然后在命令行参数中加入：

“root=/dev/vda”或许可以，如果不行，则加入"root=/dev/vdb"，并增加一个virtio-blk

```
IBM iSeries virtual disk
		  0 = /dev/iseries/vda	First virtual disk, whole disk
		  8 = /dev/iseries/vdb	Second virtual disk, whole disk
		    ...
		200 = /dev/iseries/vdz	26th virtual disk, whole disk
		208 = /dev/iseries/vdaa	27th virtual disk, whole disk
		    ...
		248 = /dev/iseries/vdaf	32nd virtual disk, whole disk
```



## 问题

- [x] 什么是IO设备？

IO设备即输入输出设备，这些设备需要OS来管理和控制，包括磁盘、串口等。

- [x] qemu启动时，这两个参数是什么意思：

```shell
-drive file=$(FS_IMG),if=none,format=raw,id=x0 \ # 将"file"文件作为一个虚拟硬盘，命名为x0
-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 # 在模拟环境中 硬盘x0会作为一个virtIO blk块设备
```

- [x] 查看qemu的dtb：

```
qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 16 \
    -m 1024 \
    -kernel ./linux/arch/arm64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01 \
    -machine dumpdtb=qemu-virt.dtb

dtc -I dtb -O dts -o qemu-virt.dts qemu-virt.dtb
```

> 