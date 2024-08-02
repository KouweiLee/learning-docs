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

## 建立IOCTL的步骤

### 1. 在driver中定义IOCTL cmd

要支持一个新的IOCTL指令, 需要如下步骤:

1. 定义一个IOCTL命令:

```c
#define "ioctl name" __IOX("magic number","command number","argument type")
```

ioctl name会表示一个int类型的值, 这个值区分IOCTL cmd.

其中IOX可以为:

“**`IO`**“: an ioctl with no parameters
“**`IOW`**“: an ioctl with write parameters (copy_from_user)
“**`IOR`**“: an ioctl with read parameters (copy_to_user)
“**`IOWR`**“: an ioctl with both write and read parameters

magic number: 区分本组IOCTL和别的IOCTL

command number: 区分组内具体的IOCTL

argument type: 传入参数的类型

例如:

```c
#include <linux/ioctl.h>
#define WR_VALUE _IOW('a','a',int32_t*)
#define RD_VALUE _IOR('a','b',int32_t*)
```

### 2. 在driver中增加IOCTL函数

```c
int  ioctl(struct file *file,unsigned int cmd,unsigned long arg)
```

这个函数会根据cmd参数来判断是哪个IOCTL, arg为用户态传入的参数. 该函数实现后需要在file_operations中的unlocked_ioctl进行指定.

### 3. 在user中定义IOCTL cmd

和在driver中定义完全一样,其实引用同一个头文件即可.

### 4. 在user中使用IOCTL 系统调用

```c
long ioctl( "file descriptor","ioctl command","Arguments");
```

参数:

<**`file descriptor`**>: This the open file on which the ioctl command needs to be executed, which would generally be device files.
<**`ioctl command`**>: ioctl command which is implemented to achieve the desired functionality
<**`arguments`**>: The arguments need to be passed to the ioctl command.

例如:

```c
ioctl(fd, WR_VALUE, (int32_t*) &number); 
ioctl(fd, RD_VALUE, (int32_t*) &value);
```



## 参考资料

https://embetronicx.com/tutorials/linux/device-drivers/ioctl-tutorial-in-linux/