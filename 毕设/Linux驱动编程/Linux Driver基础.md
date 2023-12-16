# Linux Driver基础

## 内核模块ko是什么

内核模块是一组代码，可以自由地被加载和卸载到内核。可以用于：

- Device drivers
- Filesystem drivers
- System calls

## Linux Device是什么

Linux把everything当作文件，硬件也被看作是文件。Linux的设备分为3种：

- Character device
- Block device
- Network device

## 如何编写ko模块

### Module信息

编写Module信息所需的头文件：**`Linux/module.h`** 

- License

```c
MODULE_LICENSE("GPL");
```

- Author

```c
MODULE_AUTHOR("Author");
```

- Module Description

```c
MODULE_DESCRIPTION("A sample driver");
```

- Module Version

Version的格式：`[<epoch>:]<version>[-<extra-version>]`，从前到后的版本号的优先级越低

```c
MODULE_VERSION("2:1.0");
```

### 加载和卸载函数

* 当执行insmod时，会执行：

```c
static int __init hello_world_init(void) /* Constructor */
{
    return 0;
}
module_init(hello_world_init);
```

* 当执行**`rmmod`**时，会执行：

```c
void __exit hello_world_exit(void)
{

}
module_exit(hello_world_exit);
```

### printk

内核中使用printk来打印信息，且有Log level，例如：

```
printk(KERN_INFO "Welcome To EmbeTronicX");
```

在新的内核版本中，可以使用`pr_xxx`函数来替代printk，例如：

```
pr_info – Print an info-level message. (ex. pr_info("test info message\n")).
pr_err – Print an error-level message. (ex. pr_err(“test error message\n”)).

```



## 参考资料

https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-part-1-introduction/