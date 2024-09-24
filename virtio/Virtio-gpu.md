# Virtio-gpu设备调研

## nxp关于gpu的支持

NXP包含2个GPU，一个3D GPU：GC7000UL，一个2D GPU：GC520L。

![image-20240728100242640](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240728100242640.png)

## nxp上启动gpu

OKMX8MPQ-C 支持 MIPI DSI、HDMI、LVDS 屏幕接口。在uboot选择display select，按下相应的数字，开关屏幕接口，以显示Qt界面。

### 使用gpu

* 查看是否有GPU

```
cat /sys/kernel/debug/gc/info
```

[gpu使用文档](file:///media/lgw/E/study/pku/nxp/resources/imx-yocto-L5.4.70_2.3.0/i.MX_Graphics_User's_Guide.pdf)中gpuinfo.sh位于emmc的rootfs/unit_tests/GPU中。

要使用gpu.sh，则还需要一个显示屏。

* 通过obs查看屏幕输出

打开obs，来源添加视频采集设备即可。

### gpu直通root linux

1. 修改设备树

2. smc调用如何解决？？？

### linux gpu驱动

probe函数：gpu_probe

## 物理GPU

显卡本身不是GPU，GPU是显卡的处理单元，显卡还有一个存储器，叫显存。

GPU 也可以用来作为区分 2D 硬件显卡和 3D 硬件显卡的重要依据。2D 硬件显卡主要通过使用 CPU 来处理特性和 3D 图像，将其称作 “软加速”。3D 硬件显卡则是把特性和 3D 图像的处理能力集中到硬件显卡中，也就是 “硬件加速”。

使用 GPU 有两种方式，一种是开发的应用程序通过通用的图形库接口调用 GPU 设备，另一种是 GPU 自身提供 API 编程接口，应用程序通过 GPU 提供的 API 编程接口直接调用 GPU 设备。

OpenGL（Open Graphics Library）是一个用于渲染2D和3D图形的跨语言、跨平台的应用编程接口（API）。OpenGL的主要功能是操作GPU进行高效的图形渲染，如果没有GPU，OpenGL可以使用软件渲染器（如Mesa的LLVMpipe）在CPU上执行图形渲染。

参考资料：

1. https://www.cnblogs.com/jmilkfan-fanguiju/p/11825032.html

## qemu-virtio-gpu

qemu默认提供的virtio-gpu设备是一个2D GPU，是由CPU模拟的。例如创建一个mmio virtio-gpu 2d设备，启动参数是：

```
-device virtio-gpu-device
```

guest虚拟机需要通过[Mesa](https://www.mesa3d.org/) or [SwiftShader](https://github.com/google/swiftshader)软件渲染器，利用2D GPU实现3D图形操作。

qemu的3D GPU则是由以下命令指定：

```
-device virtio-gpu-gl
```

When using virgl accelerated graphics mode in the guest, OpenGL API calls are translated into an intermediate representation (see [Gallium3D](https://www.freedesktop.org/wiki/Software/gallium/)). The intermediate representation is communicated to the host and the [virglrenderer](https://gitlab.freedesktop.org/virgl/virglrenderer/) library on the host translates the intermediate representation back to OpenGL API calls.



## virtio-gpu协议内容

virtio-gpu支持2种模式：2D和3D。3D模式下会将渲染操作交给真实GPU，因此3D模式需要一个物理3D GPU。

> scanout是个名词在这里，表示一个具体的显示输出目标，可以理解为一个显示器或显示器的一部分
>
> display则是显示器

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240731140838018.png" alt="image-20240731140838018" style="zoom:50%;" />

还没搞清楚Virtio GPU与物理GPU如何交互。



### virtqueue

virtio-gpu包含两个virtqueue：

1. control queue：用于发送控制命令
2. cursor queue：用于发送有关鼠标的控制命令

两个队列的形式相同，队列中的请求和回复都有同样的header：

```c
struct virtio_gpu_ctrl_hdr {
    le32 type;
#define VIRTIO_GPU_FLAG_FENCE (1 << 0)
#define VIRTIO_GPU_FLAG_INFO_RING_IDX (1 << 1)
    le32 flags;
    le64 fence_id;
    le32 ctx_id;
    u8 ring_idx;
    u8 padding[3];
};
    
enum virtio_gpu_ctrl_type {
    /* 2d commands */
    VIRTIO_GPU_CMD_GET_DISPLAY_INFO = 0x0100,
    VIRTIO_GPU_CMD_RESOURCE_CREATE_2D,
    VIRTIO_GPU_CMD_RESOURCE_UNREF,
    VIRTIO_GPU_CMD_SET_SCANOUT,
    VIRTIO_GPU_CMD_RESOURCE_FLUSH,
    VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D,
    VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING,
    VIRTIO_GPU_CMD_RESOURCE_DETACH_BACKING,
    VIRTIO_GPU_CMD_GET_CAPSET_INFO,
    VIRTIO_GPU_CMD_GET_CAPSET,
    VIRTIO_GPU_CMD_GET_EDID,
    VIRTIO_GPU_CMD_RESOURCE_ASSIGN_UUID,
    VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB,
    VIRTIO_GPU_CMD_SET_SCANOUT_BLOB,
    /* 3d commands */
    VIRTIO_GPU_CMD_CTX_CREATE = 0x0200,
    VIRTIO_GPU_CMD_CTX_DESTROY,
    VIRTIO_GPU_CMD_CTX_ATTACH_RESOURCE,
    VIRTIO_GPU_CMD_CTX_DETACH_RESOURCE,
    VIRTIO_GPU_CMD_RESOURCE_CREATE_3D,
    VIRTIO_GPU_CMD_TRANSFER_TO_HOST_3D,
    VIRTIO_GPU_CMD_TRANSFER_FROM_HOST_3D,
    VIRTIO_GPU_CMD_SUBMIT_3D,
    VIRTIO_GPU_CMD_RESOURCE_MAP_BLOB,
    VIRTIO_GPU_CMD_RESOURCE_UNMAP_BLOB,
    /* cursor commands */
    VIRTIO_GPU_CMD_UPDATE_CURSOR = 0x0300,
    VIRTIO_GPU_CMD_MOVE_CURSOR,
    /* success responses */
    VIRTIO_GPU_RESP_OK_NODATA = 0x1100, // 不需要返回任何数据
    VIRTIO_GPU_RESP_OK_DISPLAY_INFO,
    VIRTIO_GPU_RESP_OK_CAPSET_INFO,
    VIRTIO_GPU_RESP_OK_CAPSET,
    VIRTIO_GPU_RESP_OK_EDID,
    VIRTIO_GPU_RESP_OK_RESOURCE_UUID,
    VIRTIO_GPU_RESP_OK_MAP_INFO,
    /* error responses */
    VIRTIO_GPU_RESP_ERR_UNSPEC = 0x1200,
    VIRTIO_GPU_RESP_ERR_OUT_OF_MEMORY,
    VIRTIO_GPU_RESP_ERR_INVALID_SCANOUT_ID,
    VIRTIO_GPU_RESP_ERR_INVALID_RESOURCE_ID,
    VIRTIO_GPU_RESP_ERR_INVALID_CONTEXT_ID,
    VIRTIO_GPU_RESP_ERR_INVALID_PARAMETER,
};

```

以下依次是对驱动和设备交互中controlq的命令解释：

1. VIRTIO_GPU_CMD_GET_DISPLAY_INFO

获取当前每个scanout的信息

```c
#define VIRTIO_GPU_MAX_SCANOUTS 16
struct virtio_gpu_rect {
    le32 x; // position
    le32 y;
    le32 width; // size
    le32 height;
};
struct virtio_gpu_resp_display_info {
    struct virtio_gpu_ctrl_hdr hdr;
    struct virtio_gpu_display_one {
        struct virtio_gpu_rect r;
        le32 enabled;
        le32 flags;
    } pmodes[VIRTIO_GPU_MAX_SCANOUTS];
};

```

2. VIRTIO_GPU_CMD_RESOURCE_CREATE_2D

在host端创建一个指定宽、高、format的resource。

```
struct virtio_gpu_resource_create_2d {
    struct virtio_gpu_ctrl_hdr hdr;
    le32 resource_id;
    le32 format;
    le32 width;
    le32 height;
};

enum virtio_gpu_formats {
    VIRTIO_GPU_FORMAT_B8G8R8A8_UNORM = 1,
    VIRTIO_GPU_FORMAT_B8G8R8X8_UNORM = 2,
    VIRTIO_GPU_FORMAT_A8R8G8B8_UNORM = 3,
    VIRTIO_GPU_FORMAT_X8R8G8B8_UNORM = 4,
    VIRTIO_GPU_FORMAT_R8G8B8A8_UNORM = 67,
    VIRTIO_GPU_FORMAT_X8B8G8R8_UNORM = 68,
    VIRTIO_GPU_FORMAT_A8B8G8R8_UNORM = 121,
    VIRTIO_GPU_FORMAT_R8G8B8X8_UNORM = 134,
};
```

3. VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING

为指定的resource增加一系列backing storage，`nr_entries`表示后面跟着多少个`virtio_gpu_mem_entry`，每个`entry`表示一个内存区域。这些内存区域共同构成该resource的backing storage（也就是framebuffer），用于数据传输。

```c
struct virtio_gpu_resource_attach_backing {
    struct virtio_gpu_ctrl_hdr hdr;
    le32 resource_id;
    le32 nr_entries;
};
struct virtio_gpu_mem_entry {
    le64 addr;
    le32 length;
    le32 padding;
};
```

4. VIRTIO_GPU_CMD_SET_SCANOUT

为一次output，设置scanout的参数。

```c
struct virtio_gpu_set_scanout {
    struct virtio_gpu_ctrl_hdr hdr;
    struct virtio_gpu_rect r;
    le32 scanout_id; 
    le32 resource_id;
};
```

