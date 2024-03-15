# qemu相关

对于arm, qemu提供的物理内存RAM的起始地址为0x4000_0000

## virtio实现

文件:virtio-mmio.c, virtio.c, virtio.h, virtio-mmio.h

```c
static const MemoryRegionOps virtio_mem_ops = {
    .read = virtio_mmio_read, // 读相关函数
    .write = virtio_mmio_write, // 写相关函数
    .endianness = DEVICE_LITTLE_ENDIAN,
};
```

vdev为:

```c
struct VirtIODevice
{
    DeviceState parent_obj;
    const char *name;
    uint8_t status;
    uint8_t isr;
    uint16_t queue_sel;
    uint64_t guest_features;
    uint64_t host_features;
    uint64_t backend_features;
    size_t config_len; // config空间的长度
    void *config;
    uint16_t config_vector;
    uint32_t generation;
    int nvectors;
    VirtQueue *vq;
    MemoryListener listener;
    uint16_t device_id;
    bool vm_running;
    bool broken; /* device in invalid state, needs reset */
    bool use_disabled_flag; /* allow use of 'disable' flag when needed */
    bool disabled; /* device in temporarily disabled state */
    bool use_started;
    bool started;
    bool start_on_kick; /* when virtio 1.0 feature has not been negotiated */
    bool disable_legacy_check;
    VMChangeStateEntry *vmstate;
    char *bus_name;
    uint8_t device_endian;
    bool use_guest_notifier_mask;
    AddressSpace *dma_as;
    QLIST_HEAD(, VirtQueue) *vector_queues;
};
```

deviceclass为:

```c
struct VirtioDeviceClass {
    /*< private >*/
    DeviceClass parent;
    /*< public >*/

    /* This is what a VirtioDevice must implement */
    DeviceRealize realize;
    DeviceUnrealize unrealize;
    uint64_t (*get_features)(VirtIODevice *vdev,
                             uint64_t requested_features,
                             Error **errp);
    uint64_t (*bad_features)(VirtIODevice *vdev);
    void (*set_features)(VirtIODevice *vdev, uint64_t val);
    int (*validate_features)(VirtIODevice *vdev);
    void (*get_config)(VirtIODevice *vdev, uint8_t *config);
    void (*set_config)(VirtIODevice *vdev, const uint8_t *config);
    void (*reset)(VirtIODevice *vdev);
    void (*set_status)(VirtIODevice *vdev, uint8_t val);
    /* For transitional devices, this is a bitmap of features
     * that are only exposed on the legacy interface but not
     * the modern one.
     */
    uint64_t legacy_features;
    /* Test and clear event pending status.
     * Should be called after unmask to avoid losing events.
     * If backend does not support masking,
     * must check in frontend instead.
     */
    bool (*guest_notifier_pending)(VirtIODevice *vdev, int n);
    /* Mask/unmask events from this vq. Any events reported
     * while masked will become pending.
     * If backend does not support masking,
     * must mask in frontend instead.
     */
    void (*guest_notifier_mask)(VirtIODevice *vdev, int n, bool mask);
    int (*start_ioeventfd)(VirtIODevice *vdev);
    void (*stop_ioeventfd)(VirtIODevice *vdev);
    /* Saving and loading of a device; trying to deprecate save/load
     * use vmsd for new devices.
     */
    void (*save)(VirtIODevice *vdev, QEMUFile *f);
    int (*load)(VirtIODevice *vdev, QEMUFile *f, int version_id);
    /* Post load hook in vmsd is called early while device is processed, and
     * when VirtIODevice isn't fully initialized.  Devices should use this instead,
     * unless they specifically want to verify the migration stream as it's
     * processed, e.g. for bounds checking.
     */
    int (*post_load)(VirtIODevice *vdev);
    const VMStateDescription *vmsd;
    bool (*primary_unplug_pending)(void *opaque);
};
```

对于virtio-blk设备, class的各个函数分别为:

```c
static void virtio_blk_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    VirtioDeviceClass *vdc = VIRTIO_DEVICE_CLASS(klass);

    device_class_set_props(dc, virtio_blk_properties);
    dc->vmsd = &vmstate_virtio_blk;
    set_bit(DEVICE_CATEGORY_STORAGE, dc->categories);
    vdc->realize = virtio_blk_device_realize;
    vdc->unrealize = virtio_blk_device_unrealize;
    vdc->get_config = virtio_blk_update_config;
    vdc->set_config = virtio_blk_set_config;
    vdc->get_features = virtio_blk_get_features;
    vdc->set_status = virtio_blk_set_status;
    vdc->reset = virtio_blk_reset;
    vdc->save = virtio_blk_save_device;
    vdc->load = virtio_blk_load_device;
    vdc->start_ioeventfd = virtio_blk_data_plane_start;
    vdc->stop_ioeventfd = virtio_blk_data_plane_stop;
}
```

### virtio-blk的IO处理函数

virtio_blk_handle_vq

## qemu和kvm的关系？

* kvm是什么？具体能起什么样的作用？

Kernel-based Virtual Machine，是一个内核模块, 导出了一系列接口到用户空间, 这些接口可以创建, 配置, 启动虚拟机. KVM主要负责的是CPU虚拟化以及内存虚拟化. 

* qemu呢？

一种hypervisor，可以模拟真实的硬件。

* qemu和kvm如何配合?

qemu作为一个用户态程序, 对`/dev/kvm`进行ioctl, 即可调用kvm提供的各个控制接口. 当qemu调用执行vcpu相关的ioctl时, kvm会将执行流转换到vm的vcpu的执行流, 当vm执行一些特殊指令发生vm exit时, 执行流会陷入到kvm中, 大多数情况下kvm会转到qemu执行流, 此时ioctl返回, qemu进行判断即可. 

## linux内核源码

virtio-driver发生中断时, 会进入vm_interrupt函数

## blk设备的IO数据路程

virtio_queue_notify->

virtio_blk_handle_output->

virtio_blk_handle_vq->

virtio_blk_submit_multireq->

submit_requests->

blk_aio_pwritev->

blk_aio_prwv->

执行协程->blk_aio_read_entry

blk_co_do_preadv_part->(qemu 7.2.1)

bdrv_co_preadv_part->

bdrv_co_preadv->(drv->bdrv_co_preadv)

raw_co_preadv->(raw-format.c)

bdrv_co_preadv->

bdrv_co_preadv_part->

bdrv_aligned_preadv

bdrv_driver_preadv->

raw_co_preadv(file-posix.c)->

raw_co_prw->

raw_thread_pool_submit放入线程池

执行线程

qemu_thread_start->

worker_thread->

handle_aiocb_rw->

handle_aiocb_rw_linear(iov为单个请求)->

pread......(/preadv)

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240124102400963.png" alt="image-20240124102400963" style="zoom:50%;" />

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240124102424720.png" alt="image-20240124102424720" style="zoom:50%;" />

调试qemu的命令:

```bash
sudo gdb-multiarch --args qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -dtb hvisor.dtb \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 4  \
    -m 2G \
    -kernel linux-Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=/home/lgw/study/hypervisor/new_readme/ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -net nic \
    -net user,hostfwd=tcp::2333-:22 \
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

address_space_map函数中, flatview_translate没有进行有关IOMMU的操作, fuzz_dma_read_cb则是个空函数, 用于: 这个函数的主要目的是在进行虚拟机模糊测试时，模拟 DMA 读取操作，生成模糊测试所需的数据，并记录 DMA 区域以避免重复获取。这样可以模拟 DMA 操作对虚拟机内存的影响，用于测试虚拟机的正确性和稳定性。

***

如果增加了AIO优化编译选项的话, 可能会通过异步IO

所以可以看到, 最后是通过异步IO的方式读写文件, 其实本质还是pread和pwrite, 只不过是利用qemu中的协程和事件循环的机制实现的. 不知道DMA是什么意思???

## 内存管理

## 协程

相关资料: [qemu协程](https://blog.csdn.net/huang987246510/article/details/93139257?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-93139257-blog-8824735.235^v40^pc_relevant_3m_sort_dl_base3&spm=1001.2101.3001.4242.2&utm_relevant_index=4#t12)

## qemu通外网

qemu的网络分为两部分:

1. 虚拟网络设备, 提供给guest
2. 网络后端, 用于和虚拟网络设备, 宿主机相连

**虚拟网络设备的创建命令:**

* `- device`

```
-device TYPE,netdev=NAME
```

其中netdev为网络后端的id

* `-nic`

如果不想关注nic的太多细节, 可以采用-nic来代替-device. 比如

```
-netdev user,id=n1 -device virtio-net-pci,netdev=n1
```

```
-nic user,model=virtio-net-pci
```

* `-net`

这是传统可用但过时的用法, 比如:

```
-net nic,model=MODEL
```

命令可以查看支持的model类型

```
qemu-system-aarch64 -machine virt,gic_version=3 -net nic,model=?
```

**网络后端的创建命令:**

```
-netdev TYPE,id=NAME,...
```

 其中id是虚拟网络设备的id, 这两个对应起来会连到一块

如果TYPE为user, 即-netdev user, 那么会创建SLIRP的网络后端, 这性能很差且有限, 不建议,

TYPE为tap的是推荐使用的, 它将使用host上的tap设备.

```
-netdev tap,id=mynet0
```



## 问题

- [ ] BlockBackend对于virtio-blk是什么

