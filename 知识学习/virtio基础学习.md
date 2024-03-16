# virtio基础学习

## virtio基本概念

virtio设备是hypervisor提供的一种设备（虚拟设备），guest如果想使用这种设备，需要实现virtio设备的驱动程序。整个virtio的组织结构为：

<img src="https://camo.githubusercontent.com/c9205b73446a0ffee68b9a8665e655ad42cce287f0f433f971cc23511c2a346b/68747470733a2f2f6d6470696373346c67772e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f616c6979756e2f696d6167652d32303233313132333039353435393933312e706e67" alt="image-20231123095459931" style="zoom:80%;" />

其中driver和device之间存在一层transport层，该层利用共享内存的方式(virtqueue)进行driver和device之间的数据传输。

### guest如何发现virtio设备？

virtio根据*设备让驱动看到和访问设备的方式*，可以分为2种：

1. 基于MMIO的virtio设备：virtio设备直接挂载到系统总线上，driver通过对特定的mmio区域进行读写，就可以驱动virtio设备。
2. 基于PCI的virtio设备：virtio设备挂在到PCI总线上，用于x86。

对于Arm，virtio设备为MMIO，此时需要在设备树中提供virtio设备所在内存区域的地址、中断号等信息。若hypervisor提供了一种新的virtio设备，则需要将这些信息添加到设备树。guest解析设备树时就会发现该设备。

### virtio-driver和device之间如何交互？

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

## virtio-blk设备

首先介绍一下核心数据结构: Virtqueue

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20231227142933852.png" alt="image-20231227142933852" style="zoom:50%;" />

Virtqueue由3部分组成, 描述符表, 可用环和已用环. Queue Size由IO寄存器`QueueNum`设置. Driver负责创建Virtqueue, 并将其ipa写入QueueDescLow等多个IO寄存器，使device获知virtqueue的地址，建立guest和hypervisor之间的共享内存区域。

> buffers中的内容因不同的virtio设备而异, 这里仅以blk为例.

```c
struct virtq {
    // 描述符表
    struct virtq_desc desc[ Queue Size ];  
    // 可用环
    struct virtq_avail avail;
    // Padding to the next Queue Align boundary.
    u8 pad[ Padding ];
    // 已用环
    struct virtq_used used;
};
```

### 发生IO请求的流程

driver发出请求:

1. 首先将IO请求的命令或数据放到一个或多个buffer里. 
2. 在描述符表中分配一条描述符链, 指向这些buffer. 链头的buffer中包含IO请求的命令, 包括读或写, 起始扇区等信息. 链表中的buffers则用于存放读到或写入的数据
3. 将描述符链头的索引写入可用环, 表示有新的IO请求
4. 写notify寄存器, 通知device有新数据
5. Virtio进程被阻塞, CPU调度其他进程, 直到device发中断通知.

device执行:

1. 根据可用环中指出的描述符链, 进行具体的IO操作, 并更新结果到buffers里.
2. 将描述符链头的索引写入已用环, 表示已完成IO请求
3. 中断通知driver

driver更新已用环, 表示知悉.

### 描述符

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

`VIRTQ_AVAIL_F_NO_INTERRUPT`：driver设置这个flag，来建议device：当device处理了一个可用环中的请求后，不要发中断。但这仅仅是建议，device不一定不发中断。在以前的版本中, 这又叫做: VRING_AVAIL_F_NO_INTERRUPT

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

`VIRTQ_USED_F_NO_NOTIFY`：device通过这个flag建议driver：当driver在可用环增加一个buffer请求时，不要kick me。但只是建议。在以前的版本中, 又叫做: VRING_USED_F_NO_NOTIFY. 

### 配置空间

配置空间为不同的Virtio设备不同的IO寄存器, capacity是必须支持的, 其他根据feature选择性支持, hvisor目前支持前3个寄存器.

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

## virtio-net设备

有3种virtqueue, 顺序以此为receiveq, transmitq, controlq. 一个存放空buffer, 用来收报文；一个存放往外发送的buffer, 用来发报文；第三种队列用来控制特征.

### driver初始化

#### 协商features

VIRTIO_NET_F_CSUM (0): Device可以处理带有部分校验和的报文. 这样驱动就可以将一部分工作量卸载到设备, 减轻了主机处理这些数据包时的负担

VIRTIO_NET_F_GUEST_CSUM (1): 驱动进行部分校验数据包

VIRTIO_NET_F_GUEST_UFO (10)

VIRTIO_NET_F_HOST_TSO4 (11): device可以处理IPv4 TCP报文, 意思就是驱动可以使用卸载到设备的TCP

VIRTIO_NET_F_HOST_TSO6 (12) : device可以处理IPv6 TCP报文

VIRTIO_NET_F_HOST_UFO (14) : device可以处理UDP fragmentation报文

VIRTIO_NET_F_HOST_USO (56) : device可以处理UDP segmentation

#### 设备的配置空间

```c
struct virtio_net_config {
	u8 mac[6]; // 设备的mac地址
#define VIRTIO_NET_S_LINK_UP 1
#define VIRTIO_NET_S_ANNOUNCE 2
	le16 status;
}
```

#### 重要的初始化工作

driver初始化时会用空buffer填充receiveq. 一个buffer对应着多个描述符. 

### 传输报文

#### 发送报文

报文头对应的数据结构为:

```c
struct virtio_net_hdr {
#define VIRTIO_NET_HDR_F_NEEDS_CSUM 1
#define VIRTIO_NET_HDR_F_DATA_VALID 2
#define VIRTIO_NET_HDR_F_RSC_INFO 4
    u8 flags;
#define VIRTIO_NET_HDR_GSO_NONE 0
#define VIRTIO_NET_HDR_GSO_TCPV4 1
#define VIRTIO_NET_HDR_GSO_UDP 3    // UDP fragmentation
#define VIRTIO_NET_HDR_GSO_TCPV6 4
#define VIRTIO_NET_HDR_GSO_UDP_L4 5 // UDP segmentation
#define VIRTIO_NET_HDR_GSO_ECN 0x80
    u8 gso_type;
    le16 hdr_len;
    le16 gso_size;
    le16 csum_start;
    le16 csum_offset;
    le16 num_buffers;
};
```

由于设定了VIRTIO_NET_F_CSUM, 因此flags为VIRTIO_NET_HDR_F_NEEDS_CSUM???

gso_type表明该报文是以哪种协议传送. 

注意, 这和blk设备描述符链的组成并不相同. 报文头之后的数据格式, 是以数据链路层的格式来的:

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240109095143830.png" alt="image-20240109095143830" style="zoom:50%;" />

其中ARP报文是网络层数据报, 用于通过IP地址来获取MAC地址. 类型值为0x0806

#### 接收报文

device设置报文头num_buffers为接收这个报文用到的描述符数量, 不过由于feature的设置, num_buffers恒为1 

#### control queue的使用

driver通过control queue可以发送命令给device, 来控制一些复杂feature的使用. 命令的数据格式为:

```c
struct virtio_net_ctrl {
    u8 class;
    u8 command;
    u8 command-specific-data[];
    u8 ack;
};
/* ack values */
#define VIRTIO_NET_OK 0
#define VIRTIO_NET_ERR 1
```

其中前3个字段是driver发出, ack是device响应的

## virtio-console

linux的命令行参数中, 会指定其启动后终端对应的串口, 例如`console=ttyAMA0`则表示设备树里的第一个串口. 之后计算机会监视该串口, 如果有字符读入, 则交给shell处理解析, 比如执行一个程序. 如果有字符要向串口写, 即transmit, 则写入该串口.

因此如果想让一个虚拟机的串口操控另一个虚拟机, 那么需要每个虚拟机都有一个

* Features:

VIRTIO_CONSOLE_F_SIZE (0):Configuration cols and rows are valid.

* 设备配置空间

```c
struct virtio_console_config {
    le16 cols;
    le16 rows;
    le32 max_nr_ports;
    le32 emerg_wr;
};
```

包含4个队列:

0 receiveq(port0) : device传给driver的

1 transmitq(port0): driver发给device的. 

2 control receiveq 

3 control transmitq

## 参考资料

1. [Virtio 原理与实现-知乎](https://zhuanlan.zhihu.com/p/639301753?utm_psn=1704906158266068992)
2. [virtio设备驱动程序](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#)
3. [virtio-官方规范](https://docs.oasis-open.org/virtio/virtio/v1.1/csprd01/virtio-v1.1-csprd01.pdf)