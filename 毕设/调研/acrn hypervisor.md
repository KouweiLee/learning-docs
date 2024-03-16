# acrn hypervisor调研

## 介绍

acrn是type I型hypervisor, 运行在Intel平台上. acrn最大的特点是提供了一个Service VM, 提供IO devices, 其他User VM可以使用Service VM提供的devices. 

* 设备模拟

ACRN提供了3种设备模拟的策略:

1. 模拟设备: 在Service VM中实现了全虚拟化和半虚拟化的虚拟设备. 其过程和咱们之前的类似. non root->hv->root ko->root user->hv->non root
2. 直通设备: 将设备直通给User VM
3. Mediated passthrough device: 这种设备是前两者的结合, 与性能相关较大的资源会直通给User VM, 例如data plane；而其他的control plane会进行模拟. 

 <img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240125095722004.png" alt="image-20240125095722004" style="zoom: 80%;" />





## device model

![image-20240123102037501](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240123102037501.png)

request buffer的设计:

For each VM, there is a shared 4-KByte memory region used for I/O request communication between the hypervisor and Service VM. Upon initialization of a VM, the DM (`acrn-dm`) in the Service VM userland first allocates a 4-KByte page and passes the GPA of the buffer to the hypervisor via hypercall. The buffer is used as an array of 16 I/O request slots with each I/O request being 256 bytes. This array is indexed by vCPU ID. **Thus, each vCPU of the VM corresponds to one I/O request slot in the request buffer since a vCPU cannot issue multiple I/O requests at the same time**.

IO request的处理流程:

![image-20240123104635644](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240123104635644.png)

提高性能:

Efficient: batching operation is encouraged

Batching operation and deferred notification are important to achieve high-performance I/O, since notification between the FE driver and BE driver usually involves an expensive exit of the guest. Therefore batching operating and notification suppression are highly encouraged if possible. This will give an efficient implementation for performance-critical devices.

### 主事件循环

为了监控各个fd, acrn采用mevent来检测fd. mevent的主循环函数为mevent_dispatch. 当最初执行main函数时, 会跳转到mevent_dispatch, 直到关闭hypervisor. 在mevent_dispatch中, 会不停调用epoll_wait, 当有监测的fd可用时, 则会进入mevent_handle函数调用run函数进行处理. 

## virtio device

![image-20240123170831206](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240123170831206.png)

![image-20240123171106964](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240123171106964.png)

和之前咱们设想的几乎一模一样. 

### kernel land

acrn在Linux kernel中还实现了vhost device

### virtio blk的实现

* 初始化

vm_init_vdevs函数初始化所有的虚拟设备-->

init_pci-->

pci_emul_init-->

*ops*->vdev_init-->

virtio_blk_init

每个virtio blk device都会创建8个work线程, 用来处理读写请求. 线程执行的函数为blockif_thr. 具体进行文件读写的函数位于blockif_proc. 使用了preadv, pwritev和psync. 

* 处理virtio请求

virtio_blk_proc则是virtio blk处理virtio请求的主体函数. 

### virtio net的实现

* 初始化

命令行指定:

```
Add a PCI slot to the Device Model acrn-dm command line (mac address is optional):
-s 4,virtio-net,tap=<name>,[mac=<XX:XX:XX:XX:XX:XX>]
```

virtio_ops: virtio_net_ops

virtio_net_init: 初始化net设备

​	--> virtio_net_tap_setup: 创建并启动tap设备. 

​		-->mevent_add: 将tap fd加入到epoll监测的列表中, 回调函数设置为virtio_net_rx_callback

​	--> 创建virtio_net_tx_thread线程,

* 处理请求

tx queue的notify函数: virtio_net_ping_txq

    virtio_net_ping_txq -->       // start TX thread to process, notify thread return
      virtio_net_tx_thread -->   // this is TX thread
        virtio_net_proctx -->      // call corresponding backend (tap) to process
            virtio_net_tap_tx -->
                writev -->                  // write data to tap device

rx 收到报文后, 中断会发给hypervisor, hypervisor再进行中断注入, 然后epoll_wait的fd会返回, 之后在mevent_handle调用之前注册的回调函数virtio_net_rx_callback:

```
virtio_net_rx_callback-->
virtio_net_tap_rx--> 调用readv, 从tapfd中读取数据
--> 向guest发出中断
```



* 结构

可参考 [这里](https://projectacrn.github.io/latest/developer-guides/hld/virtio-net.html). 数据流就是, virtio net device要与外界通信时, 将数据发给TAP设备, TAP再发给bridge, bridge最终发给物理nic. 反着来就是接收外界的数据包.

![image-20240207105033877](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240207105033877.png)



## 问题

- [ ] 针对DM来说, 用了什么优化没有. 性能如何???
- [ ] 搞清楚DMA是什么意思在acrn的图里
- [ ] virtio-blk是否用到了写缓存, 这个怎么实现. 