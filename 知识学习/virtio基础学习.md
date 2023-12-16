# virtio基础学习

## virtio是什么？

virtio设备是hypervisor提供的一种设备（虚拟设备），guest如果想使用这种设备，需要实现virtio设备的驱动程序。整个virtio的组织结构为：

<img src="https://camo.githubusercontent.com/c9205b73446a0ffee68b9a8665e655ad42cce287f0f433f971cc23511c2a346b/68747470733a2f2f6d6470696373346c67772e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f616c6979756e2f696d6167652d32303233313132333039353435393933312e706e67" alt="image-20231123095459931" style="zoom: 67%;" />

其中driver和device之间存在一层transport层，该层利用共享内存的方式(virtqueue)进行driver和device之间的数据传输。

## guest如何发现virtio设备？

virtio根据*设备让驱动看到和访问设备的方式*，可以分为2种：

1. 基于MMIO的virtio设备：virtio设备直接挂载到系统总线上，driver通过对特定的mmio区域进行读写，就可以驱动virtio设备。
2. 基于PCI的virtio设备：virtio设备挂在到PCI总线上，用于x86。

对于Arm，virtio设备为MMIO，此时需要在设备树中提供virtio设备所在内存区域的地址、中断号等信息。若hypervisor提供了一种新的virtio设备，则需要将这些信息添加到设备树。guest解析设备树时就会发现该设备。

## virtio-driver和device之间如何交互？

* 读写配置空间发生data abort

设备树提供的virtio mmio内存区域称为**配置空间**。guest没有配置空间的stage 2地址映射，当driver读写配置空间，就会导致data abort，陷入el2。hypervisor即可根据driver读写的内容和寄存器进行处理。

* driver通知device进行IO请求

driver**写QueueNotify寄存器**，发生data abort后el2便可得知有数据请求，device可以将该请求下放到真实的物理设备，或进行仿真模拟。

* device通知driver完成IO请求

device完成IO请求后，有2种方式通知driver：

1. 中断注入：将设备树中virtio设备的中断号对应的**中断注入**guest。
2. 直接返回结果：driver通过**轮询**的方式确定device是否完成io操作。

* 共享内存virtqueue的实现

最初，driver会读写配置空间的寄存器与device进行协商。此时driver会为virtqueue分配内存空间，并将其ipa写入QueueDescLow等多个IO寄存器，使device获知virtqueue的地址，这样便建立了guest和hypervisor之间的共享内存区域。

## blk：核心数据结构

### 配置空间

```c
struct virtio_blk_config {
    le64 capacity;
    le32 size_max;
    le32 seg_max;
    struct virtio_blk_geometry {
        le16 cylinders;
        u8 heads;
        u8 sectors;
    } geometry;
    le32 blk_size;
    struct virtio_blk_topology {
        // # of logical blocks per physical block (log2)
        u8 physical_block_exp;
        // offset of first aligned logical block
        u8 alignment_offset;
        // suggested minimum I/O size in blocks
        le16 min_io_size;
        // optimal (suggested maximum) I/O size in blocks
        le32 opt_io_size;
    } topology;
    u8 writeback;
    u8 unused0[3];
    le32 max_discard_sectors;
    le32 max_discard_seg;
    le32 discard_sector_alignment;
    le32 max_write_zeroes_sectors;
    le32 max_write_zeroes_seg;
    u8 write_zeroes_may_unmap;
    u8 unused1[3];
};
```

### 一个描述符链的组成

* 描述符的结构

```c
struct virtq_desc {
    /* Address (guest-physical). */
    le64 addr;
    /* Length. */
    le32 len;
    /* The flags as indicated later. */
    le16 flags;
    /* Next field if flags & NEXT */
    le16 next;
};
```

flags分有：

```c
/* This marks a buffer as continuing via the next field. */
#define VIRTQ_DESC_F_NEXT 1
/* This marks a buffer as device write-only (otherwise device read-only). */
#define VIRTQ_DESC_F_WRITE 2
/* This means the buffer contains a list of buffer descriptors. */
#define VIRTQ_DESC_F_INDIRECT 4
```

* 描述符链头所在的buffer的结构：

```c
le32 type;
le32 reserved;
le64 sector;
```

其中sector表示第几个扇区，即乘上512B就是以字节为单位的偏移量。type分为：

```
In = 0, // read
Out = 1,// write
Flush = 4, // save device cache to real device
GetId = 8,
GetLifetime = 10,
Discard = 11,
WriteZeroes = 13,
SecureErase = 14,
```

* 描述符链数据buffer的结构

```
u8 data[][512];
```

* 描述符链尾buffer的结构

```c
u8 status
```

status可分为：

```c
#define VIRTIO_BLK_S_OK 0 // success
#define VIRTIO_BLK_S_IOERR 1 //error
#define VIRTIO_BLK_S_UNSUPP 2// not supported request
```

### 可用环

```c
struct virtq_avail {
#define VIRTQ_AVAIL_F_NO_INTERRUPT 1
    le16 flags;
    le16 idx;
    le16 ring[ /* Queue Size */ ];
    le16 used_event; /* Only if VIRTIO_F_EVENT_IDX */
};
```

`VIRTQ_AVAIL_F_NO_INTERRUPT`：driver设置这个flag，来建议device：当device处理了一个可用环中的请求后，不要发中断。但这仅仅是建议，device不一定不发中断。

### 已用环

```c
struct virtq_used {
#define VIRTQ_USED_F_NO_NOTIFY 1
    le16 flags; 
    le16 idx;
    struct virtq_used_elem ring[ /* Queue Size */];
    le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */
};

/* le32 is used here for ids for padding reasons. */
struct virtq_used_elem {
    /* Index of start of used descriptor chain. */
    le32 id;
    /* Total length of the descriptor chain which was used (written to) */
    le32 len;
};
```

`VIRTQ_USED_F_NO_NOTIFY`：device通过这个flag建议driver：当driver在可用环增加一个buffer请求时，不要kick me。但只是建议。

## 参考资料

1. [Virtio 原理与实现-知乎](https://zhuanlan.zhihu.com/p/639301753?utm_psn=1704906158266068992)
2. [virtio设备驱动程序](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#)
3. [virtio-官方规范](https://docs.oasis-open.org/virtio/virtio/v1.1/csprd01/virtio-v1.1-csprd01.pdf)