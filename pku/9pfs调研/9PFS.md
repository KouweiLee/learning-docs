# 9PFS

在QEMU中使用9PFS，可以使guest OS访问host上指定的目录，该目录被guest OS视为一个虚拟文件系统设备（virtio-9p-device）。host和guest之间的通信采用9P（Plan 9 Filesystem Protocol），guest OS可将其当做直通设备。

## 如何使用9pfs

### 准备

下载linux内核代码，构建内核镜像时配置一些选项。

下载qemu，将内核镜像安装进去，并确保kvm模块已经被load。

### 启动guest

启动guest时，要在qemu命令行上加入以下命令来开启QEMU的9P sharing：

```
    -virtfs FSDRIVER,path=PATH_TO_SHARE,mount_tag=MOUNT_TAG,security_model=mapped|mapped-xattr|mapped-file|passthrough|none[,id=ID][,writeout=immediate][,readonly][,fmode=FMODE][,dmode=DMODE][,multidevs=remap|forbid|warn][,socket=SOCKET|sock_fd=SOCK_FD]
```

这个命令可以实现：

FSDRIVER：设置要使用的文件系统驱动后端，推荐local，QEMU直接在host上调用VFS函数

TRANSPORT_DRIVER：host和guest之间的通信驱动，一般为virtio-9p-pci

path(path to share)：为文件系统设备设置export path. host上的path路径上的文件会被共享给guest上的9p client

security_model：为path设置安全模式。mapped-xattr|mapped-file|passthrough|none，推荐mapped-xattr。它将文件的属性（uid、gid、mode bits）被存储为文件属性（元数据），可以防止guest修改

readonly：如果设置，guest将只能读path

mount_tag：设定在guest中export point的名字

其中最重要的就是：path、mount_tag、security_model

### 挂载shared path

在启动后的guest上运行指令：

```
mount -t 9p -o trans=virtio [mount tag] [mount point] -oversion=9p2000.L
```

mount_tag就是之前设定的，mount_point就是guest上shared path的路径。这里使用virtio作为传输方法。

- [ ] virtio 协议和9p 协议有什么区别

## 9P protocol

9pfs使用Plan 9 Filesystem Protocol在guest和host之间进行文件IO通信。

9pfs的基本结构为：

![9pfs topology.png](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309011459320.png)

其中在QEMU中的有3个组件：9p server、9p filesystem driver、9p transport dirvers

* 9p client：是guest oS需要提供的，最常见的是[Linux 9p client](https://github.com/torvalds/linux/tree/master/fs/9p) (在linux源码的fs文件夹下可以找到，与ramfs在同一文件夹下)

* 9p server：9pfs的控制器部分，处理原始的9p network protocol handling，以及从9p client发来的9p request的高层次控制流。其代码大部分位于[9p server][https://gitlab.com/qemu-project/qemu/-/blob/master/hw/9pfs/9p.c]，4000多行C

* 9p filesystem driver：9p server使用 VFS 层(Virtual file system)进行实际的文件操作，有3种实现，最常用和最推荐的是local fs driver：

它只是将各个VFS函数直接映射到host的文件系统函数（open, read, write, etc）。具体实现位于[hw/9pfs/9p-local.c](https://gitlab.com/qemu-project/qemu/-/blob/master/hw/9pfs/9p-local.c)。

* 9p transport dirvers：host和guest之间的通信通道。例如：

virtio transport driver：使用虚拟PCI设备和virtio协议在guest和host之间传输9p信息，代码见[hw/9pfs/virtio-9p-device.c](https://gitlab.com/qemu-project/qemu/-/blob/master/hw/9pfs/virtio-9p-device.c)。

## 线程和协程

QEMU 中的9pfs 大量使用 Coroutines 来处理单个9p request. 

协程仅仅和堆栈有关，它管理一个自己的堆栈。线程之间可以互相传递协程，以协作、顺序地完成一个任务。

9pfs中，控制流以及线程和协程的关系的示意图如下：

![9pfs control flow.png](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309011635717.png)

每个9p client request都会对应一个协程，当server端的主线程处理协程时，发现需要由fs driver调用IO操作时（可能阻塞一段时间），就将其交给工作线程。工作线程执行完后又会将协程交还给主线程。

几乎整个9p server都运行在QEMU主线程上，一些工作线程用来处理fs driver 文件IO任务。因此在[9p.c][https://gitlab.com/qemu-project/qemu/-/blob/master/hw/9pfs/9p.c]中，`*_co_*()`形式的函数会将协程发给工作线程，当该函数返回时，协程被返送给主线程。

## arceos在启动时设置环境变量

arceos启动时，在rust_main函数中，设置环境变量前就对文件系统进行了初始化。根据文件系统中指定的文件，从文件系统中读取环境变量并设置。

## 将9p 作为root fs

9pfs可以作为guest的整个文件系统，这样guest的所有文件其实都是host上的文件。

优点：



## 参考资料

1. 使用9pfs：[Documentation/9psetup - QEMU](https://wiki.qemu.org/Documentation/9psetup)

2. 开发者文档：[Documentation/9p - QEMU](https://wiki.qemu.org/Documentation/9p)

3. 将9p作为root fs：[Documentation/9p root fs - QEMU](https://wiki.qemu.org/Documentation/9p_root_fs)

4. Virtual file system：[Virtual file system - Wikipedia](https://en.wikipedia.org/wiki/Virtual_file_system)

> VFS是在真实文件系统和内核之间的一个抽象层，允许应用程序以统一的方式访问不同类型的文件系统。

5. 介绍9p 协议的论文：[Using 9P2000 Under Linux](https://www.usenix.org/legacy/events/usenix05/tech/freenix/full_papers/hensbergen/hensbergen.pdf)



## TODO&进展

- [x] 阅读使用9pfs：[Documentation/9psetup - QEMU](https://wiki.qemu.org/Documentation/9psetup)
- [x] 阅读开发者文档：[Documentation/9p - QEMU](https://wiki.qemu.org/Documentation/9p)
- [ ] ~~阅读将9p作为root fs：[Documentation/9p root fs - QEMU](https://wiki.qemu.org/Documentation/9p_root_fs)~~

- [x] 阅读[v9fs: Plan 9 Resource Sharing for Linux — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/9p.html)
- [ ] 9p protocol传输的接口是什么
- [ ] 9p protocol[9P (protocol) - Wikipedia](https://en.wikipedia.org/wiki/9P_(protocol))
