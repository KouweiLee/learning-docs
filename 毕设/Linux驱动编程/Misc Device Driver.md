# Misc Device Driver

杂项驱动设备, 用于编写一个简单的驱动设备, 该设备特点为:

1. 无需分配主设备号. 默认的主设备号为10, 从设备号可以自定义为1到255. 该设备会出现在`/dev/{your_misc_file}`
2. 不需要调用**`cdev_init`**, **`cdev_add`**, **`class_create`**, and **`device_create`**. 会自动创建device node和device file

## Misc Driver API

头文件: **`include<linux/miscdevice.h>`**.

### Misc Device结构体

```c
struct miscdevice {
  int minor;
  const char *name;
  struct file_operations *fops;
  struct miscdevice *next, *prev;
};
```

minor: 建议使用**`MISC_DYNAMIC_MINOR`**动态分配从设备号

name: misc driver的名称,会出现在/dev/文件夹下

fops: 与character device中的fops相同

next, prev: 用来管理环形链表

### 注册misc device

```c
int misc_registerstruct miscdevice * misc)
```

misc: 要注册的misc device

返回值: 0表示成功

该函数可以避免调用许多cdev, class, device的初始化和创建函数

### 卸载misc device

```c
misc_deregister(struct miscdevice * misc)
```

## 参考资料

https://embetronicx.com/tutorials/linux/device-drivers/misc-device-driver/