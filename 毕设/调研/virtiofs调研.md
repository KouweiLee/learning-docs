# virtiofs调研

## FUSE

使用FUSE编写一个用户态文件系统的教程: https://blog.csdn.net/wenwen091100304/article/details/48011813. 其大致做法是, 调用libfuse库中的fuse_main, 将支持的操作集合`fuse_operations`以及挂载点比如`/tmp/fuse_eg`传给fuse_main, 这就会自动创建一个FUSE用户态的文件系统, 并挂载到`/tmp/fuse_eg`下. 

### 参考资料

FUSE协议:https://man7.org/linux/man-pages/man4/fuse.4.html

https://fishpi.cn/article/1640420458343?p=1&m=0

https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/fuse.h

通过libfuse编写FUSE文件系统的教程: https://blog.csdn.net/t3swing/article/details/82901628

daemon和FUSE.ko的工作流程介绍: https://zhuanlan.zhihu.com/p/143256077. 简单来讲, 就是作为daemon，文件系统进程通过读取"/dev/fuse"设备文件来获得数据，如果pending queue为空，它将陷入阻塞等待. 当pending queue上有请求到来时，daemon进程将被唤醒并处理这些请求。

## virtiofs和FUSE的关系



## 工作流程

## 好处

对于rhyper, 通过virtiofs可以让多个non root共享root上的一个目录. non root上有virtiofs的driver, root自然放上virtiofs的device(即daemon文件系统)

为了支持在不同guest中同时mmap(MAP_SHARED)同一个文件，virtio-fs把文件mmap进qemu的进程地址空间并让不同guest使用DAX访问该内存空间，这样就绕过了guest pagecache达到不同guest都访问同一份数据的目的，同时也在多个guest之间共享了内存，节省了内存资源。

virtiofs的目标: 在多个虚拟机之间提供共享的本地文件系统, 从而实现共享一个目录树. 

## virtiofs的具体实现

driver类似与fuse中的kernel module fuse.ko, device为fuse daemon守护进程. 通过virtqueue, 根据fuse协议进行通信.

fuse协议: https://man7.org/linux/man-pages/man4/fuse.4.html 

virtqueues包含: 

- 0 hiprio
- 1 notification queue (only VIRTIO_FS_F_NOTIFICATION feature is set)
- 2...n request queues 

hiprio用于存放高优先级的fuse请求(FUSE_INTERRUPT, FUSE_FORGET, and FUSE_BATCH_FORGET), 避免request queues已满导致无法发送高优先级请求. 

request queues之间没有顺序, 多个队列只是为了提高性能. 

### 设备配置空间

```c
struct virtio_fs_config {
    char tag[36]; //文件系统名
    le32 num_request_queues;
    le32 notify_buf_size;
};
```

### request queue

存放在request queue中的fuse request格式满足fuse协议规定, 即:

```c
struct virtio_fs_req {
    // Device-readable part
    struct fuse_in_header in;
    u8 datain[];
    // Device-writable part
    struct fuse_out_header out;
    u8 dataout[];
};
```

