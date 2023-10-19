# multiboot

[Multiboot Specification version 0.6.96 (gnu.org)](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)

multiboot规范了引导程序和操作系统之间的接口，使得引导程序可以引导任何符合规范的操作系统。主要用于PC，如x86。

服从所有被标记为“must”的引导程序或者OS映像被称为符合Multiboot规范。

* 几个基本概念

*OS image*

*boot module*：和OS image一起被bootloader载入内核的辅助文件

## 规范要求

一个OS image必须包含一个multiboot header，必须在前8192个字节当中被包含。这个header的布局必须为：

![image-20230909094729314](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309090947659.png)

### boot information format

进入操作系统时，EBX寄存器包含：32位物理地址，指向multiboot information structure，它包含了Bootloader传递给OS的重要信息，其格式为：

![image-20230909095257999](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309090952114.png)

如果flags[2]被设定，那么cmdline中包含cmd line的物理地址。cmd line是一个C类型的字符串，以\0结尾。