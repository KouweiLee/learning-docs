# rhyper-源码分析

**rhyper采用的virtio规范为1.1版本**

## virtio device

rhyper实现的两种virtio device，virtio-blk和virtio-net，在收到driver的请求后，都会将其添加到一个异步任务队列，并向MVM发送SGI（或称IPI）核间中断，MVM读取异步任务后会将其下放到真实设备（直通给MVM）处理，处理完成后再通知GVM。流程可见：

![image-20231124104643782](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20231124104643782.png)

### virtio-blk device

rhyper实现了一个完整的virtio-blk device，主要代码位于`src/device/virtio/blk.rs`。其初始化是在vmm_init_emulated_device函数：

* emu_register_dev：注册设备的异常处理函数`emu_virtio_mmio_handler`
* emu_virtio_mmio_init：进行一系列数据结构初始化

当el1访问配置空间时，发生data abort，此时函数调用流程为：

lower_aarch64_synchronous-->

data_abort_handler-->

emu_handler-->

**emu_virtio_mmio_handler**-->：该函数会根据访问的配置空间寄存器作出相应反应

notify_handler-->

call_notify_handler-->

virtio_blk_notify_handler：该函数中会将IO请求添加到异步任务队列，并向MVM发送核间中断。MVM会读取任务并下放到真实的disk，处理完成后会通过中断注入通知GVM。

### virtio-net device

rhyper的设计理念类似微内核，MVM连接真实的net设备，可以与外网通信。其他GVM要想与外网通信，可以将数据通过rhyper内部的virtio-net设备发送给MVM，由MVM发送给外网。当MVM从外网收到报文时，根据其mac地址就可以转发给指定VM。即virtio-net设备用于内网通信，MVM承担着交换机的作用。

其初始化与blk类似，不过发生data abort后的异常处理函数有所不同：

...-->

call_notify_handler-->

virtio_net_notify_handler（队列index不为2）

virtio_net_handle_ctrl（index为2）

## virtio-blk driver

rhyper实现了一个简单的virtio-blk设备的驱动程序，目测是可以通过这个driver与qemu上的virtio-blk设备进行数据传输。代码位于src/driver/

## virtio在vm之间是如何通知和传递的？

### driver向device发请求

virtio-blk的数据请求处理函数：

virtio_mediated_blk_notify_handler-->

add_async_task-->：增加ipi task

async_task_exe--> poll task，会执行async_ipi_req函数：

async_ipi_req-->

ipi_send_msg-->：向mvm发送ipi（只给0号CPU发）

```rust
ipi_send_msg(0, IpiType::IpiTMediatedDev, IpiInnerMsg::MediatedMsg(msg));
```

然后执行流返回到guest vm。（对于sysHyper，应该不用异步任务）

***

当vm发生中断时，会陷入EL2，执行interrupt_handler，查询对应类型的处理函数:

对于INTERRUPT_IRQ_IPI(1)，则处理函数为ipi_irq_handler:

其中又会根据Ipi的类型，查询不同的处理函数。对于IpiTMediatedDev，则为

mediated_ipi_handler-->

* virtio_blk_notify_handler-->

  * generate_blk_req：

    * 在IO_LIST中加入一个IO 请求


    * 更新used info到ASYNC_USED_INFO_LIST

* finish_async_task：将ipi任务从队列中拿出

* async_task_exe：执行IO task，执行async_blk_io_req函数

async_blk_io_req-->

mediated_blk_read-->

hvc_send_msg_to_vm-->

hvc_guest_notify：中断注入到mvm, mvm对数据请求进行处理。中断注入的vcpu是当前cpu的vcpu.

hvc时执行的函数：hvc_ivc_handler，其中会执行对应ipi type的handler函数。其中，IpiTHvc会执行**hvc_ipi_handler**

发生ipi后，目标CPU会执行的处理函数ipi_irq_handler

### device向driver返回请求

vm通过hvc，执行hvc_mediated_handler-->

mediated_blk_notify_handler

### mvm和hv对于blk共享的数据结构

其中mvm和hypervisor间对blk设备共享的数据结构为：

```rust
#[repr(C)]
pub struct MediatedBlkContent {
    // 请求次数
    nreq: usize,
    cfg: MediatedBlkCfg,
    req: MediatedBlkReq,
}

#[repr(C)]
pub struct MediatedBlkCfg {
    name: [u8; 32],
    block_dev_path: [u8; 32],
    block_num: usize,
    dma_block_max: usize,
    cache_size: usize,
    idx: u16,
    // TODO: enable page cache
    pcache: bool,
    cache_va: usize,
    cache_ipa: usize,
    cache_pa: usize,
}

#[repr(C)]
pub struct MediatedBlkReq {
    req_type: u32,
    sector: usize,
    count: usize,
}
```

其中MediatedBlkContent的地址由该结构体给出：

```rust
#[derive(Clone)]
pub struct MediatedBlk {
    // MediatedBlkContent的内存地址
    pub base_addr: usize, 
    pub avail: bool, // mediated blk will not be removed after append
}
```

* 共享数据结构的约定

hvc的所有处理函数中，包含一个hvc_mediated_handler，处理vm发出的有关mediated设备的处理函数。其中：

mediated_dev_append：将传入的ipa，ipa实际上就是MediatedBlkContent的ipa，给EL2.

mediated_blk_notify_handler：MVM处理完数据请求，--》async_task_exe-->finish_async_task-->AsyncIoTask的分支，进行中断注入，如果是读操作则从cache复制到数据缓冲区

### 总结与思考

rhyper其实就是，先发ipi让mvm的一个cpu陷入el2，然后el2将IO请求放在共享的数据结构里以便vm能获取到，然后中断注入到mvm，让mvm在el1或0来处理这个请求。



## 问题

- [x] 为什么rhyper要使用异步队列？？？要不自己设计一下？

rhyper的异步任务跟同步的一样，主动调用的poll函数，这就直接执行了。sysHyper不需要异步任务应该

- [x] 共享数据结构具体的地址约定这些怎么搞
- [x] HVC_IRQ是什么？

在pi4_def.rs中，mvm_config_init函数里有一个device名为vm_service的，其中断号就是HVC_IRQ。其类型为EmuDeviceTShyper，还注册到了设备树里面，异常处理函数是什么？？？

通过ko文件弄得

## 参考资料

1. [rust_shyper](https://gitee.com/openeuler/rust_shyper/tree/master)