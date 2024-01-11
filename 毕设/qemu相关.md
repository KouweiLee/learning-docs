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

Kernel-based Virtual Machine，是Linux内核的一部分，用于实现硬件虚拟化，可以提高qemu的性能

* qemu呢？

一种hypervisor，可以模拟真实的硬件。

## linux内核源码

virtio-driver发生中断时, 会进入vm_interrupt函数