# virtio实现

## 路线设计

### 1. non root 发出virtio 请求

对于非数据请求，进程需要等待返回结果。

对于数据请求，进程会被挂起，直到设备发出中断通知EL0或EL1,此时中断处理函数会执行相关操作。

### 2. EL2跳板收到virtio请求

将请求封装成结构体EmuContext: 访问地址、读还是写、寄存器编号、寄存器的值。

放入链表VIRTIO-REQ中。或者map也行，key为cell的id。

通知root cell：发中断SGI。（考虑多个VM同时发SGI的情况，怎么办???）

**对于非数据请求，需要阻塞吧，等待结果???**

对于数据请求，则可以返回

***

root cell的0号CPU陷入EL2, 从VIRTIO-REQ中取出请求，放到共享的数据结构里。

然后中断注入0号CPU（rhyper通知的是0号CPU对应的vcpu），0号CPU在EL1下先从共享的数据结构中处理。

### 3. root cell处理请求

首先root cell在最开始应该创建一个守护进程，例如：

```
sudo tools/shyper system daemon [mediated-cfg.json] &
```

其中`&`表示将该进程放入后台执行。

## 开发日志

* 目前需要注意的

0号cpu必须给root cell独占

el2页表注意考虑虚拟地址到物理地址的映射，如果没有则得建立。

命名规范：函数命名目前与Jailhouse统一; 数据结构命名统称为hvisor_device

* 这周争取完成的工作

- [ ] 1. 写一个共享数据结构，使得linux el1和el2能共享数据
  2. 学习信号机制，看如何让el1和el0共享请求和数据。
  3. 实现el1中断处理函数，参照linux内核源码
  4. 让el0能拿到数据请求。
  5. 测试一下，看看能不能拿到，简单测试测试。

### 12.9

- [x] 中断注入到root cell，看处理函数怎么弄

需要ko文件弄，需要学习。

- [ ] 着手写ko文件，关键是写一个中断处理函数，这是最关键的。没用的别学了

### 12.11

内核与用户的通信：

内核向用户进程发送信号，信号可以携带一个int的信息

一个int不够啊。Linux内核和用户态程序能否共享一片内存区域，用mmap。

目前两种方案：

信号+mmap:可以参考：https://www.cnblogs.com/LiuYanYGZ/p/14247489.html

~~信号+sysfs，由sysfs file的show函数让内核给用户程序传递参数。~~

siginfo的一些域：

```c
#define si_value	_sifields._rt._sigval
#define si_int		_sifields._rt._sigval.sival_int
#define si_ptr		_sifields._rt._sigval.sival_ptr
```

- [x] 再看看sysfs

### 12.14

着手实现el2和el1的共享内存。

考虑并发问题：（之后再考虑 ）

1. 多个cell同时给root发请求：root最多支持同时的4个virtio req请求，超过4个则可能出现问题。之后再解决
2. 同个cell的多个cpu给root发请求：每个请求要记录原始cpu

**目标，先打通整个路径，不考虑cache; 先只考虑单核单vm的情况，不考虑多的**

- [x] el2的页表，没有device_address物理地址的虚拟地址映射，看怎么解决。

映射一个呗。

先把路线整个走通，然后自己写配置文件？？？

规定一下传给用户态virtio配置文件的json格式：

```json
{

}
```



## 问题

- [x] rhyper是怎么把virtio请求下放给真实的设备

 rhyper会将virtio请求在el2转化为一个自定义的请求，交给mvm. mvm收到后，会根据真实设备驱动（直通）将数据读写到virtio-dev的缓冲区cache里，然后经由el2将该缓冲区cache再复制到virtioqueue里。

- [x] rhyper VM的virtio-blk-device是什么到底？

按这张图来看，就是root cell中的一个文件。

<img src="https://zmhqcp62hu.feishu.cn/space/api/box/stream/download/asynccode/?code=MzJjODBiMzBlY2M4NmEyNWUzYThmZDEyNTE2MTIwMzdfeGpVcmozeUlqc3o5Q0FIWkxRQ1I3Y2c1Zk02eWhBYm9fVG9rZW46V3dBZWJldW42b2FmVVB4M1lIV2NOeVNpbkFoXzE3MDEzMzQwMjM6MTcwMTMzNzYyM19WNA" alt="img" style="zoom:50%;" />

- [x] rhyper VM的virtio-net device是什么？

mvm直通了一个net设备，通过这个net设备就可以上外网. 然后mvm收到外网来的报文，根据mac地址通过virtio-net发送给指定的vm。

rhyper类似微内核，virtio-device实现内网通信。

- [x] 内存隔离的问题。root cell的el2应该得能访问non root给出的数据缓冲区，这是必须的。因此得看下EL2页表的映射

可以访问。所有cpu的el2页表是一样的，而且没有动过在初始化之后，除了映射tmp区域。可**能需要建立个虚拟地址到物理地址的映射。**

- [x] 从qemu扒还是自己写？感觉自己写吧，不过可以参考qemu.
- [x] cpu阻塞，如何阻塞？还得判断是哪个位置的

suspend一下就行