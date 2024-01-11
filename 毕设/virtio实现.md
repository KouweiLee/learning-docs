# virtio实现

## net实现

- [ ] 确定device端要怎么实现

虚拟机之间的通信是好说的, 直接在rx_vq上写数据就行

虚拟机和外网之间的通信如何弄??? non root如何识别root是一个路由器? 只要给定ip地址就行应该. 

- [ ] net的那些features的作用搞清楚, 看怎么实现

## 跳板机制的改进

我们现在面临的需求是:

多个不同设备的请求发生, 必须都能及时处理.

同一个设备的请求连续发生, 必须都能及时处理.

要保证el0和el2共享内存的互斥性和一致性.    

越简单越好. 

现在的几种方案是:

1. 共享内存有一个请求总数, 还有一个已经处理的总数, el0可以判断已经处理的总数和请求总数的差距, 来进行请求的处理. 可以采用环形队列来存放请求. 

   关键是队列满了怎么办. 如果队列满了, 那么将virtio端的请求队列阻塞, 直到队列不满. 

   具体做法是, 设置一个原子变量, 如果队列满了, 将该原子变量设为1, 等完成一个请求后, 就将该原子变量设为0. 表示不满.  

   也可以采用信号量的PV操作, 向VIRTIOREQLIST增加一个请求, P一下, 完成一个请求, 就V一下. 感觉这种最好了. 

   **不用了, 在handle_virtio_requests中如果发现环形队列满了,那么就不再往里加请求了, 等finish req的hvc发生时,再调用这个函数判断是否有新的请求发生.** 

2.  还是沿用现在的做法, 每个队列有一个req的位置, 太复杂了.......

- [ ] ~~内核发送signal时的sig_int信息可以保存req的位置, req组成一个数组. 每个队列有一个req的位置. req的位置可以是由共享内存的一个字段实现, el1负责读, el2负责写, 而且~~

  简单一点, 共享内存有一个请求总数, 还有一个已经处理的总数, el0可以判断已经处理的总数和请求总数的差距, 来进行请求的处理. 可以采用环形队列来存放请求. 

  先暂定这个做法, 感觉这个做法更加合适. 不会丢失任何mmio访问. 

- [ ] 确保notify请求不会丢失. 要不就取消el2的判断, 
- [ ] 是否有可能, 同时对一个设备的多个队列进行操作. 

***



- [x] 当el0进行nreq的处理时, el2发来一个请求, 怎么保证这段内存的使用是互斥的, 而且是互不影响的. 

按理来说只有一个设备的时候, 不应该有这个问题. 因为virtio进程应该是顺序进行的吧? 阻塞之后, 他就没法再访问了

- [x] **linux发生数据请求返回后, 好像没有阻塞???????????**

```
[INFO  3] hvisor::device::virtio:42] notify !!!, cpu id is 3
[INFO  3] hvisor::device::virtio:68] non root returns
[INFO  0] hvisor::device::emu:111] req is 0
[INFO  0] hvisor::device::emu:45] push req, nreq is 1
[INFO  3] hvisor::device::virtio:42] notify !!!, cpu id is 3
[INFO  3] hvisor::device::virtio:68] non root returns
[INFO  0] hvisor::device::emu:111] req is 1
[INFO  0] hvisor::device::emu:45] push req, nreq is 2
[INFO  3] hvisor::device::virtio:75] notify resolved
[INFO  0] hvisor::device::emu:111] req is 1
[INFO  0] hvisor::device::emu:45] push req, nreq is 2
[INFO  2] hvisor::device::virtio:42] notify !!!, cpu id is 2
[INFO  2] hvisor::device::virtio:68] non root returns
[INFO  0] hvisor::device::emu:111] req is 0
[INFO  0] hvisor::device::emu:45] push req, nreq is 1
[INFO  2] hvisor::device::virtio:75] notify resolved
```

- [x] 感觉得看看, 研究一下通信过程, 可能要把请求先组合起来, 再进行数据传输.

## blk实现

不要管legacy. 不需要改成cpp. 

- [ ] 这个是什么意思?

```
    config->size_max = BLK_SIZE_MAX;
    config->seg_max = BLK_SEG_MAX;
```

seg_max好像是一个描述符链中描述符个数的最大值

- [x] 整virtq
- [x] 分配linux的ram给non root的时候, 为什么root执行free时内存大小不变?

root可用的内存, 通过命令行指定的. 

- [x] 能否分配linux的ram给non root时, 不取消root的映射

能

- [x] 检查一下queue num 是否为256, 如果是的话, 那么512的大小是不是开大了

改成512了

- [x] used_flags等初始化
- [x] img读出来的应该是一样的
- [x] 目前猜测是non root的启动参数root没写对, 看看原来启动磁盘文件系统是怎么启动的

不是, 有行代码写串了

- [x] 打开linux的debug功能,以输出更多信息

貌似只能一个一个文件的打开

- [x] make clean解决dummy问题

## 地址映射

当发生queue的notify时, root发出hvc, 请求地址映射, ipa应该是一个固定的地址, 映射到真实的pa. 然后root通过mmap, 将用户态的地址映射到ipa, 然后这样进行数据请求.

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

- [x] 1. 写一个共享数据结构，使得linux el1和el2能共享数据
  2. 学习信号机制，看如何让el1和el0共享请求和数据。
  3. 实现el1中断处理函数，参照linux内核源码
  4. 让el0能拿到数据请求。
  5. 测试一下，看看能不能拿到，简单测试测试。

### 12.9

- [x] 中断注入到root cell，看处理函数怎么弄

需要ko文件弄，需要学习。

- [x] 着手写ko文件，关键是写一个中断处理函数，这是最关键的。没用的别学了

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

### 12.18

- [x] user先实现mmap
- [x] user实现signal handler
- [x] driver实现Mmap
- [x] driver实现signal
- [x] user实现IOCTL到driver到el2

### 12.19

打通non root 和 root. 记得给non root的设备树中增加virtio的区域

以后不再记录每天实现了什么, 而是一个个路线以任务列表的形式写下来, 主要要知道自己在干什么, 按顺序写.

## 路线完成度

0x0a003e00, 大小为0x1000, 为virtio区域

root的el2和el1的共享内存区域包含两个, 一个是请求的, 一个是结果的.

el2, el1, el0是全部共享的

- [x] user先实现mmap
- [x] user实现signal handler
- [x] driver实现Mmap
- [x] driver实现signal
- [x] user实现IOCTL到driver到el2
- [x] 打通non root和root, 使两者之间能互相发请求和结果
  - [x] non root向root发送请求
  - [x] root向non root返回请求
- [x] 找到问题了,SGI可能冲突了

- [x] 吧makefile改了, 加上编译hvisor_usr和ko

```
[   68.815498] ------------[ cut here ]------------
[   68.815869] Unexpected interrupt received!
[   68.816140] WARNING: CPU: 0 PID: 0 at drivers/irqchip/irq-gic-v3.c:650 gic_handle_irq+0x124/0x158
[   68.816165] Modules linked in: main(O) jailhouse(O)
[   68.816275] CPU: 0 PID: 0 Comm: swapper/0 Tainted: G        W  O      5.4.0 #1
[   68.816303] Hardware name: linux,dummy-virt (DT)
[   68.816349] pstate: 60000085 (nZCv daIf -PAN -UAO)
[   68.816381] pc : gic_handle_irq+0x124/0x158
[   68.816412] lr : gic_handle_irq+0x124/0x158
[   68.816435] sp : ffff800010003fe0
[   68.816461] x29: ffff800010003fe0 x28: ffffc8c7fa9a2d80 
[   68.816508] x27: 0000000000000000 x26: ffff800010004000 
[   68.816553] x25: ffff800010000000 x24: 0000000000000000 
[   68.816597] x23: 0000000060000005 x22: ffffc8c7f9287680 
[   68.816642] x21: ffffc8c7fa993ef0 x20: 000000000000002a 
[   68.816687] x19: 000000000000002a x18: 0000000000000000 
[   68.816732] x17: 0000000000000000 x16: 0000000000000000 
[   68.816777] x15: ffffc8c7fab71fb0 x14: ffffc8c7fa9b2448 
[   68.816822] x13: 0000000000000000 x12: ffffc8c7fab71000 
[   68.816867] x11: ffffc8c7fa9b2000 x10: ffffc8c7fab71600 
[   68.816911] x9 : 0000000000000000 x8 : ffffc8c7fab71000 
[   68.816955] x7 : 0000000000000004 x6 : 00000000000001b2 
[   68.816999] x5 : ffff00002cea0180 x4 : 0000000000000001 
[   68.817043] x3 : ffff00002cea0180 x2 : 0000000000000000 
[   68.817087] x1 : 0000000000000000 x0 : ffffc8c7fa9a2d80 
[   68.817134] Call trace:
[   68.817293]  gic_handle_irq+0x124/0x158
[   68.817326]  el1_irq+0xb8/0x180
[   68.817358]  arch_cpu_idle+0x10/0x20
[   68.817393]  cpu_startup_entry+0x20/0x40
[   68.817433]  rest_init+0xd4/0xe0
[   68.817479]  arch_call_rest_init+0xc/0x14
[   68.817510]  start_kernel+0x418/0x448
[   68.817542] ---[ end trace cf6d7b16ba20af6f ]---


// 增加enable_irq:
[   86.861835] Unbalanced enable for IRQ 42
[   86.862740] WARNING: CPU: 10 PID: 523 at kernel/irq/manage.c:605 __enable_irq+0x44/0x6c
[   86.862840] Modules linked in: main(O+) jailhouse(O)
[   86.863840] CPU: 10 PID: 523 Comm: insmod Tainted: G           O      5.4.0 #1
[   86.863893] Hardware name: linux,dummy-virt (DT)
[   86.864448] pstate: 60000085 (nZCv daIf -PAN -UAO)
[   86.864517] pc : __enable_irq+0x44/0x6c
[   86.864562] lr : __enable_irq+0x44/0x6c
[   86.864610] sp : ffff80001050bb00
[   86.864651] x29: ffff80001050bb00 x28: 0000000000000100 
[   86.864759] x27: ffffa078b2775f30 x26: ffffa078a58b7080 
[   86.864800] x25: 0000000000000002 x24: ffff8000109c5000 
[   86.864839] x23: ffff0000200e5e80 x22: 0000000000000000 
[   86.864877] x21: ffffa078a58b7000 x20: 000000000000002a 
[   86.864916] x19: ffff00002b872600 x18: 0000000000000020 
[   86.864954] x17: 0000000000000000 x16: ffffa078b2742490 
[   86.864993] x15: ffffa078b3f71fb2 x14: ffffa078b3db2448 
[   86.865031] x13: 0000000000000000 x12: ffffa078b3f71000 
[   86.865070] x11: ffffa078b3db2000 x10: ffffa078b3f71600 
[   86.865487] x9 : 0000000000000000 x8 : ffffa078b3f71000 
[   86.865531] x7 : 0000000000000004 x6 : 0000000000000184 
[   86.865569] x5 : ffff00002cf7c180 x4 : 0000000000000001 
[   86.865618] x3 : ffff00002cf7c180 x2 : 0000000000000000 
[   86.865661] x1 : 0000000000000000 x0 : ffff000020038000 
[   86.865930] Call trace:
[   86.866072]  __enable_irq+0x44/0x6c
[   86.866132]  enable_irq+0x44/0x94
[   86.867308]  hvisor_init+0x94/0x1000 [main]
[   86.867346]  do_one_initcall+0x4c/0x190
[   86.867378]  do_init_module+0x50/0x250
[   86.867405]  load_module+0x1ed8/0x2530
[   86.867432]  __do_sys_finit_module+0xd4/0xf0
[   86.867458]  __arm64_sys_finit_module+0x1c/0x24
[   86.867612]  el0_svc_common.constprop.0+0x68/0x160
[   86.867639]  el0_svc_handler+0x20/0x80
[   86.867665]  el0_svc+0x8/0xc
[   86.867906] ---[ end trace acd3e9645feec579 ]---
[   86.869215] hvisor init done!!!

```

- [x] 为linux增加一个叫做vm_service的设备, 放入dts. 参考rhyper. 

- [x] 看看一个进程收到信号后会干什么, 处理完信号呢?

只有内核态返回用户态时,才会检查进程是否有信号发来. 如果发来, 则执行处理函数, 处理函数执行完, 再回到原来的地方执行. 如果此时信号发生多次, 则之后只会执行一次.

- [x] mmap分配的地址要按页面对齐
- [x] 整理代码, 增加注释, push 到dev分支, 进行整合


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