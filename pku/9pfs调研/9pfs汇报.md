# 9pfs

## 9pfs是什么

9pfs，是一种文件系统，在QEMU中使用9PFS，可以允许guest访问host上指定的目录，该目录会被guest OS视为一个虚拟文件系统设备（virtio-9p-device）。host和guest之间的通信（文件IO、协商等等）采用Plan 9(9p)协议。

* 9pfs的整体结构

9pfs的整体结构为：

![image-20230906150938640](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309061509817.png)

9pfs在QEMU中有3个模块：**9p server**、9p filesystem driver、9p transport dirvers。Guest OS则需要实现一个**9p client**。

9p client发出创建、删除、读取、写入文件的请求，会由qemu中的9p server进行处理。server和client之间的消息通信采用9p协议。

## 9p协议

9p协议，全称The Plan 9 File Protocol，被用来规范9p client和server之间的消息通讯。

* 消息通信的流程

消息通信的流程是，client向server发送requests(T-message)，随后server向client返回replies(R-message)。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309061519218.png" alt="image-20230906151936114" style="zoom:67%;" />

* 消息的格式

消息的格式为：消息头部是固定的7个字节，前4个字节表示整个消息的长度。第5个字节表示消息的类型（在request_create时设定）。后两个字节表示消息的标签tag，用来验证，R-message和T-message中tag应该相同。之后的字节序列时根据不同类型的消息不定长的，

属于消息的数据内容，它前两个字节表示数据的长度n，之后的n个字节均是数据。字符串用UTF-8编码，不以空字符结尾，空字符在9p中是违法的。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309061523250.png" alt="image-20230906152346152" style="zoom:67%;" />

## virtio设备

由于QEMU实际上是给guest呈现一个virtio设备，因此有必要了解什么是virtio设备。参考教程有很多，在文末的参考资料中。

Virtio是一种前后端架构，包括前端驱动（Guest内部）、后端设备（QEMU设备）、传输协议（vring），其架构如下图所示：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309062106712.png" alt="image-20230906210603539" style="zoom: 33%;" />

virtio设备支持三种设备呈现模式，其中包括：

Virtio Over MMIO，虚拟设备直接挂载到系统总线上

Virtio Over PCI BUS，遵循PCI规范，挂在到PCI总线上，作为virtio-pci设备呈现，在QEMU虚拟的x86计算机上采用的是这种模式，unikraft和qemu之间也是采用PCI进行通信的。

Virtio设备的模拟就是通过QEMU完成的，QEMU在虚拟机启动之前，会创建虚拟设备。虚拟机启动后检测到设备，调用内核的virtio设备驱动程序来加载这个virtio设备。对于KVM虚拟机，每个KVM虚拟机都是一个QEMU进程，虚拟机的virtio设备是QEMU进程模拟的，虚拟机的内存也是从QEMU进程的地址空间内分配的。

虚拟机和QEMU之间的通信，有两个部分：

1. 消息通知机制

虚拟机产生IO请求后，会通知QEMU。具体方式是通过写PCI配置空间的寄存器。具体而言，我们为虚拟机创建的virtio设备都是PCI设备，它们挂在PCI总线上，遵循通用PCI设备的发现、挂载等机制。当虚拟机启动发现virtio PCI设备时，只有配置空间可以被访问，它的形式如下图所示：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309062115621.png" alt="img" style="zoom:80%;" />

下面的kick机制就是读写配置空间上指定的寄存器，当QEMU检测到kick时，就会进行相应的处理。

2. 数据共享机制

VRING是由虚拟机virtio设备驱动创建的用于数据传输的共享内存，QEMU进程通过这块共享内存获取前端设备递交的IO请求。

## 9pfs in unikraft

9pfs在unikraft中共分为5层：

```c
c语言标准库函数
---------------
vfscore
---------------
9pfs
---------------
uk9p
---------------
plat/virtio
```

下图则更详细地展示了9pfs在unikraft中是如何工作的：

![image-20230905152938995](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309061534384.png)

> 图中实线箭头表示该结构体中存在某个域指向另一个结构体。
>
> vfscore_file->dentry：该文件对应的目录项
>
> dentry->mount：该目录项所在的挂载点
>
> dentry->vnode：该dentry对应的vnode，描述该目录项所代表文件的各种属性
>
> mount->dentry: 挂载路径对应的dentry和该挂载文件系统的根目录的dentry
>
> vnode->mount:该vnode所在的挂载点
>
> 9pdev->virtio 9pdev：9pdev对应的virtio层的9pdev
>
> 9preq中各个buf的作用
>
> xmit_buf：发送缓冲区，存放message的头部
>
> recv_buf:接收缓冲区，接收message的头部
>
> zc_buf：零拷贝缓冲区，即需要外部来读写的缓冲区。就是实际存放文件数据的，对应下面的data[count]
>
> ```
> size[4] Rread tag[2] count[4] data[count]
> size[4] Twrite tag[2] fid[4] offset[8] count[4] data[count]
> ```

### vfscore

在库函数和9pfs两层之间，是`vfscore` (*Virtual File System Core*)，即VFS虚拟文件系统层，它是9pfs和应用程序的接口层，提供了对文件系统管理的通用系统调用接口。其中有四个非常重要的数据结构：

1. `vfscore_file`：文件对象结构体，是VFS对一个文件的描述，在文件被打开时创建，包含文件描述符fd。
2. `dentry`：目录项结构体
3. `mount`：挂载结构体。挂载的作用，就是将一个设备（存储设备）挂接到一个已知目录下，访问该目录就是访问该存储设备。`mount`中`m_op`域指向该文件系统的基本操作，包括mount，unmount，只要具体的文件系统实现这些接口，就可以被注册到VFS中统一管理和使用。
4. `vnode`：文件信息管理结构体，用于管理文件的具体信息。

### 9pfs&uk9p

在virtio层之上的9pfs实现。几个重要的结构体包括：

1. `uk_9pdev`：9pdev是一个专门用来与9p设备进行通信的结构体，其中priv指向*virtio_9p_device*
2. `uk_9p_file_data`：对应一个9p文件
3. `uk_9pfs_node_data`：对应上层vfs中vnode的结构体，即`vnode->v_data`
4. `uk_9pfs_mount_data`：用来与上层vfs mount绑定的数据，即`mount->m_data`

5. `uk_9preq`：该结构体描述一个9P request，通过uk_9pdev_req_create()来创建。
6. `uk_9pfid`：该结构体是一个fid的wrapper，里面还包含的qid（server qid）

### virtio

virtio层则是与qemu进行数据传输的最底层。下图是IO请求经过virtio、qemu直到硬件的过程。

<img src="https://pic3.zhimg.com/80/v2-238882f028ba2301b36ef74cd41460b2_1440w.webp" alt="img"  />

几个重要的结构体包括：

1. `virtio_9p_device`：virtio层中，表示9p的device。包含scatter/gather list、virtqueue、mount tag等重要属性。

## 9pfs in unikraft源码分析

### virtio-9p设备初始化

TODO：virtio-9p初始化具体是在哪里调用的还需探索

9pfs的作用就是能将host上指定的目录共享给guest。那么首先QEMU需要在命令行中进行配置，指定要共享的目录路径（path），以及这个路径对应的mount tag。之后guest会通过mount tag来对virtio devices进行识别，从而发现qemu模拟的9pfs。

当unikraft发现一个virtio-9p device时???，virtio_driver会执行virtio_9p_add_dev函数。在这个函数中，会根据virtio_dev创建相应的virtio_9p_device，并读取PCI配置空间中的mount tag，并保存到tag域中。

```c
VIRTIO_BUS_REGISTER_DRIVER(&v9p_drv);
static struct virtio_driver v9p_drv = {
	.dev_ids = v9p_dev_id,
	.init    = virtio_9p_drv_init, // 主要初始化了uk_9pdev_trans
	.add_dev = virtio_9p_add_dev 
};
```

### 挂载流程

将9pfs挂载到unikraft，最初是调用一个系统调用函数：

```c
UK_SYSCALL_R_DEFINE(int, mount, const char*, dev, const char*, dir, const char*, fsname, unsigned long, flags, const void*, data)
```

这个注册宏会产生一个名为*uk_syscall_r_mount*的系统调用函数，其中dev参数就表示mount tag，guest通过该tag会找到对应的virtio device。

**TODO：哪里调用**

在该函数中，打开了名为dev的device，创建了挂载结构体`mount`，并调用了`VFS_MOUNT`宏执行了挂载函数`uk_9pfs_mount`：

```c
static int uk_9pfs_mount(struct mount *mp, const char *dev, 
int flags __unused, const void *data)
```

该函数主要流程为：

1. 创建`uk_9pfs_mount_data md`，并根据data给md的proto，uname,aname字段赋值。

2. 调用`uk_9pdev_connect`函数，这个函数会返回一个9pdev，并且这个dev拥有一些传输相关的操作（connect，request），并会与底层virtio_9p_dev进行连接（通过`virtio_9p_connect`函数实现）

3. 调用`uk_9p_version`函数，与9p server开启一段会话，主要是与server协商9p协议版本号。该过程首先会创建一个类型为TVERSION的9p request，并根据9p 协议为该request写入相关内容，并执行`send_and_wait_no_zc`函数。这个函数的细节详见接收和发送request章节。

4. 执行`uk_9p_attach`函数，与9p server建立连接。该函数将afid、用户名、要使用的文件系统名等通过9p request发给server，并创建一个9pfs根目录的fid。

   > afid的作用和解释：Permission to attach to the service is proven by providing a special fid, called `afid`, in the `attach` message. This `afid` is established by exchanging `auth` messages and subsequently manipulated using `read` and `write` messages to exchange authentication information not defined explicitly by 9P. Once the authentication protocol is complete, the `afid` is presented in the `attach` to permit the user to access the service.

### 9p request相关流程

以一个标准库函数read为例，其定义如下：

```c
ssize_t read(int fd, void*buf, size_t count)
// 在unikraft中，实际定义为UK_SYSCALL_R_DEFINE(ssize_t, read, int, fd, void *, buf, size_t, count)
```

其调用路线图如下：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309062139631.png" alt="image-20230906213953422" style="zoom:67%;" />

到`uk_9p_read`函数时，它会依次调用如下的函数：

#### 1. request_create

```c
req = request_create(dev, UK_9P_TREAD);
```

该函数会创建一个type为TREAD的request。之后会调用`uk_9preq_write`函数，向req.buf中写入一部分消息头信息。消息头信息的格式见：[intro(9P)](https://9fans.github.io/plan9port/man/man9/intro.html)，之后会调用一个最关键的函数：`send_and_wait_zc`

#### 2. send_and_wait_zc

该函数依次执行：

* uk_9preq_ready：将req标记为准备状态，初始化req的前7个固定字节，以及zc_buf相关的
* uk_9pdev_request：向virtio层发送request，virtio层将其相应的buf缓冲区通过调用`virtqueue_buffer_enqueue`放入VRING（即add_buf操作），并通过notify函数写virtio-9pfs设备对应pci配置空间的特定寄存器，指示virtqueue有请求。这时qemu就会工作，当其完成操作后，会通过中断来通知unikraft。
* uk_9preq_waitreply：忙等qemu的中断通知

当qemu发出pci中断时，对应的中断处理函数为：`virtio_pci_handle`。该函数会最终执行virtqueue的callback函数，实际上就是`virtio_9p_recv`。

## 名词示意

当guest要挂载os时，可能会遇到如下的名词：

1. mount tag

mount_tag is the tag associated by the server to each of the exported mount points. Each 9P export is seen by the client as a virtio device with an associated "mount_tag" property. Available mount tags can be seen by reading `/sys/bus/virtio/drivers/9pnet_virtio/virtio<n>/mount_tag `files.

2. uname

user name to attempt mount as on the remote server. The server may override or ignore this value. Certain user names may require authentication.

3. aname

aname specifies the file tree to access when the server is offering several exported file systems.

## 参考资料

* 9pfs

[intro(9P) - Plan 9 from User Space (9fans.github.io)](https://9fans.github.io/plan9port/man/man9/intro.html)：介绍9p协议和消息格式

[Documentation/9psetup - QEMU](https://wiki.qemu.org/Documentation/9psetup)：如何在qemu和linux虚拟机上使用9pfs

[Documentation/9p - QEMU](https://wiki.qemu.org/Documentation/9p)：9pfs在qemu上是怎么工作的

[v9fs: Plan 9 Resource Sharing for Linux — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/9p.html)：mount的一些选项有解释

* virtio设备：

[virtio设备驱动程序 - rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.cn)](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html)

[网络虚拟化 virtio 框架分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/542483879)

* [unikraft文件系统](https://tenonos.github.io/categories/#collapse%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E4%B8%8E%E5%AD%98%E5%82%A8)：

[vfscore核心数据结构 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/23/vfscore核心数据结构/)

[9p相关基础知识 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/9p基础知识/)

[unikraft文件系统导览 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/unikraft文件系统导览/)

[9pfs相关结构以及源码分析 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/9pfs/#2-1-9preq-init-9preq-c)：最重要的

[9pfs kvm qemu源码流程 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/11/07/9pfs kvm qemu源码流程/#unikraft侧)：讲了当qemu完成9p操作后，如何发通知给unikraft的

* unikraft相关：

[Hack Series: Behind the Scenes - Unikraft](https://unikraft.org/guides/hackathon-behind-scenes)：讲了怎么启动9pfs。

## TODO

- [ ] unikraft，是如何mount 9pfs，和初始化virtio-9pfs的。
- [x] virtio-9p确实是用PCI的

