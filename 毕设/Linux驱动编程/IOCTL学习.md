# IOCTL in Linux (I/O Control) 

内核和用户进程之间的通信方式有：

- **IOCTL**
- Procfs
- [Sysfs](https://embetronicx.com/tutorials/linux/device-drivers/sysfs-in-linux-kernel/)
- Configfs
- Debugfs
- Sysctl
- UDP Sockets
- Netlink Sockets

本文主要介绍IOCTL的通信方式。

IOCTL全称**Input and Output Control**，用于用户程序和设备驱动进行通信。

## 参考资料

https://embetronicx.com/tutorials/linux/device-drivers/ioctl-tutorial-in-linux/