# 在nxp上跑hvisor

Linux可用的物理内存是`0x40000000-0xCfffffff`

启动板子命令:

```
setenv mmcroot /dev/mmcblk1p1 rootwait rw;setenv fdt_file OK8MP-C-root.dtb; run mmcargs; ext4load mmc 1:1 ${loadaddr} home/arm64/OK8MP-linux-kernel/arch/arm64/boot/Image;ext4load mmc 1:1 ${fdt_addr} home/arm64/OK8MP-linux-kernel/arch/arm64/boot/dts/freescale/OK8MP-C-root.dtb; booti ${loadaddr} - ${fdt_addr}
```

## 串口

### jailhouse

在setup.c中, init_early函数调用arch_dbg_write_init:

设置arch_dbg_write为uart_write-->uart_write_char

```
struct uart_chip uart_imx_ops = {
	.init = uart_init,
	.is_busy = uart_is_busy,
	.write_char = uart_write_char,
};
```

## 用jlink调试

### 硬件准备

首先按下图将nxp板子和jlink相连接：

nxp板子：

![image-20240325112839235](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405250900714.png)

jlink：

![image-20240325112911745](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405250900787.png)

### 下载JLink

首先在Jlink官网下载[JLink](https://www.segger.com/downloads/jlink/), 找到下图位置下载即可：

![image-20240325113430951](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405250900744.png)

安装：

```
sudo dpkg -i 文件名.deb
```

执行JLinkExe测试是否安装成功。

### 使用JLinkGDBServer调试

命令行执行：

```
JLinkGDBServer -select USB -if JTAG -device Cortex-A53
```

然后GDB执行：

```
gdb-multiarch \
	-ex 'file $(target_elf)' \
	-ex 'target remote:2331' 
```

注意，JLink竟然只能调1个核。

## 挂载设备

SD 卡具体是多少，还得看linux的启动输出

![image-20240525092600870](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405250926956.png)

## 制作sd卡

1. 将SD卡插入读卡器，并连接至主机。

2. 执行以下命令，进行分区：

   ```bash
   fdisk <$DRIVE> # DRIVE可以是/dev/sdb，注意没有1
   d  # 删除所有分区
   n  # 创建新分区
   p  # 选择主分区
   1  # 分区编号为1
   16384  # 起始扇区
   回车
   t  # 更改分区类型
   83  # 选择Linux文件系统（ext4）
   w  # 保存并退出
   ```

3. 将启动文件写入SD卡启动盘：

   ```bash
   dd if=imx-boot_1G.bin of=<$DRIVE> bs=1K seek=32 conv=fsync
   ```

4. 格式化SD卡启动盘的第一个分区为ext4格式：

   ```bash
   mkfs.ext4 <$DRIVE>1
   ```

5. 将rootfs1.ext4烧到sd卡中：

```
sudo dd if=filesystem.img of=/dev/sdb1 bs=4M status=progress
```

## 启动命令

```
setenv serverip 192.169.137.2; setenv ipaddr 192.169.137.3; setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0x50000000; setenv zone0_fdt_addr 0x70000000; tftp ${loadaddr} ${serverip}:hvisor.bin; tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb; tftp ${zone0_kernel_addr} ${serverip}:Image; tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

只有hvisor.bin：

```
setenv serverip 192.169.137.2; setenv ipaddr 192.169.137.3; setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0x50000000; setenv zone0_fdt_addr 0x70000000; ext4load mmc 1:1 ${loadaddr} /home/arm64/hvisor.bin; ext4load mmc 1:1 ${fdt_addr} /home/arm64/OK8MP-C.dtb; ext4load mmc 1:1 ${zone0_kernel_addr} /home/arm64/Image; ext4load mmc 1:1 ${zone0_fdt_addr} /home/arm64/linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

新的：

```
setenv serverip 192.169.137.2; setenv ipaddr 192.169.137.3; setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0xa0000000; setenv zone0_fdt_addr 0xb0000000; tftp ${loadaddr} ${serverip}:hvisor.bin; tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb; ext4load mmc 1:1 ${zone0_kernel_addr} /home/arm64/Image; tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```



### 启动non root

```
insmod hvisor.ko
./hvisor zone start --kernel Image2,addr=0x50000000 --dtb linux2.dtb,addr=0x70000000 --id 1
```

起始地址a0000000,dtb放在b0000000

```
nohup ./hvisor virtio start \
	--device blk,addr=0xa003c00,len=0x200,irq=78,zone_id=1,img=rootfs2.ext4 &
```



## TODO

- [x] 修改hvisor的写死的地址. 

- [x] flush掉dcache, 在arch_entry一开始的时候

- [x] 解决这个报错:

  ```
                  /* RAM 00*/ {
                          .phys_start = 0x40000000,
                          .virt_start = 0x40000000,
                          .size = 0x80000000,
                          .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                  JAILHOUSE_MEM_EXECUTE,
                  },
                  /* Inmate memory */{
                          .phys_start = 0x60000000,
                          .virt_start = 0x60000000,
                          .size = 0x10000000,
                          .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
                                  JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_DMA,
                  },
  ```

- [x] 确保enable hypervisor的页表后, 能平滑过渡

- [ ] 目前gic的init是可以的, 但是hv page table的enable是不行的. 不知道为啥........

