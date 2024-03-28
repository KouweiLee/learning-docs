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

![image-20240325112839235](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240325112839235.png)

jlink：

![image-20240325112911745](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240325112911745.png)

### 下载JLink

首先在Jlink官网下载[JLink](https://www.segger.com/downloads/jlink/), 找到下图位置下载即可：

![image-20240325113430951](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240325113430951.png)

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