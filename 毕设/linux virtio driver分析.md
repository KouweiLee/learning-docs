# linux virtio driver分析

本文主要聚焦于linux virtio driver的初始化以及与qemu device的通信过程. 

## driver初始化

* virtio_mmio_probe

位于virtio_mmio.c, 探测virtio 设备, 并初始化写入VIRTIO_CONFIG_S_ACKNOWLEDGE表示发现了该设备.

调用了device_add(位于core.c), 没什么用.

在virtio_mmio.c中, 针对mmio的ops:

```c
static const struct virtio_config_ops virtio_mmio_config_ops = {
	.get		= vm_get,
	.set		= vm_set,
	.generation	= vm_generation,
	.get_status	= vm_get_status,
	.set_status	= vm_set_status,
	.reset		= vm_reset,
	.find_vqs	= vm_find_vqs,
	.del_vqs	= vm_del_vqs,
	.get_features	= vm_get_features,
	.finalize_features = vm_finalize_features,
	.bus_name	= vm_bus_name,
};
```

* virtio_dev_probe

位于virtio.c, 使用对应的driver来初始化该设备. 首先标记VIRTIO_CONFIG_S_DRIVER, 表明找到合适的driver. 又设置VIRTIO_CONFIG_S_FEATURES_OK.

而后调用了virtio_driver的probe函数, 对于virtio_blk, 则是virtblk_probe. 对于net设备, 则为virtnet_probe. 

该函数结束时, virtio设备的状态应该是VIRTIO_CONFIG_S_DRIVER_OK

```c
static struct bus_type virtio_bus = {
	.name  = "virtio",
	.match = virtio_dev_match,
	.dev_groups = virtio_dev_groups,
	.uevent = virtio_uevent,
	.probe = virtio_dev_probe,
	.remove = virtio_dev_remove,
};
```

* virtblk_probe

设置seg_max等属性.

调用init_vq-->

-->virtio_find_vqs: 设定该vq的callback函数为virtblk_done, 当设备完成IO操作后发中断时, 中断处理函数会调用该callback函数. 

-->vm_find_vqs: 注册了IO请求完成后中断处理函数为vm_interrupt. 

​	vm_setup_vq: 设置vq配置空间, 

​		vring_create_virtqueue_split, **设置vq->vq.callback为virtblk_done,vq->notify为vm_notify.** 并设置desc table, 已用环可用环给device

之后再进行一些feature的协商, 并标记VIRTIO_CONFIG_S_DRIVER_OK.

在virtio_blk.c中, 针对virtio blk的ops:

```c
static struct virtio_driver virtio_blk = {
	.feature_table			= features,
	.feature_table_size		= ARRAY_SIZE(features),
	.feature_table_legacy		= features_legacy,
	.feature_table_size_legacy	= ARRAY_SIZE(features_legacy),
	.driver.name			= KBUILD_MODNAME,
	.driver.owner			= THIS_MODULE,
	.id_table			= id_table,
	.probe				= virtblk_probe,
	.remove				= virtblk_remove,
	.config_changed			= virtblk_config_changed,
#ifdef CONFIG_PM_SLEEP
	.freeze				= virtblk_freeze,
	.restore			= virtblk_restore,
#endif
};
```



函数virtio_mmio_probe, virtio_mmio.c

-->register_virtio_device

-->device_add, core.c

sb_getblk: 

```c
static inline struct buffer_head *
sb_getblk(struct super_block *sb, sector_t block)
{
	return __getblk_gfp(sb->s_bdev, block, sb->s_blocksize, __GFP_MOVABLE);
}
```

__getblk_gfp:

```c
/*
 * __getblk_gfp() will locate (and, if necessary, create) the buffer_head
 * which corresponds to the passed block_device, block and size. The
 * returned buffer has its reference count incremented.
 *
 * __getblk_gfp() will lock up the machine if grow_dev_page's
 * try_to_free_buffers() attempt is failing.  FIXME, perhaps?
 */
struct buffer_head *
__getblk_gfp(struct block_device *bdev, sector_t block,
	     unsigned size, gfp_t gfp)
{
	struct buffer_head *bh = __find_get_block(bdev, block, size);

	might_sleep();
	if (bh == NULL)
		bh = __getblk_slow(bdev, block, size, gfp);
	return bh;
}
```

__find_get_block:

mount_root: 挂载根文件系统, 已经开始了该函数



virtqueue增加请求: 

virtblk_add_req: 

```c
static int virtblk_add_req(struct virtqueue *vq, struct virtblk_req *vbr,
		struct scatterlist *data_sg, bool have_data)
{
	struct scatterlist hdr, status, *sgs[3];
	unsigned int num_out = 0, num_in = 0;
	// 初始化一个scatterlist数组, 只有一个元素, 指向out_hdr
	sg_init_one(&hdr, &vbr->out_hdr, sizeof(vbr->out_hdr));
	sgs[num_out++] = &hdr;

	if (have_data) {
		if (vbr->out_hdr.type & cpu_to_virtio32(vq->vdev, VIRTIO_BLK_T_OUT))
			sgs[num_out++] = data_sg;
		else
			sgs[num_out + num_in++] = data_sg;
	}

	sg_init_one(&status, &vbr->status, sizeof(vbr->status));
	sgs[num_out + num_in++] = &status;

	return virtqueue_add_sgs(vq, sgs, num_out, num_in, vbr, GFP_ATOMIC);
}
```

```
virtqueue_add_split
```

最新进展: desc table这些基地址是正确的

~~但是取消映射后, 竟然root没有报错, 只是Mmio报了错, 仔细看看这块为什么明天.~~: 重新看了一下, 报错了

**qemu启动root linux命令行中的mem=768M, 正好规定了linux可以使用的内存区域大小为3000_0000, 因此不会和non root linux的内存区域发生冲突, 而且还可以访问non root linux的内存区域**

最新进展: root 可以访问到non root的内存区域. 

重点关注一下: __do_execve_file

kernel_init-->

## IO请求

### driver发出请求

```c
static const struct blk_mq_ops virtio_mq_ops = {
	.queue_rq	= virtio_queue_rq,
	.commit_rqs	= virtio_commit_rqs,
	.complete	= virtblk_request_done,
	.init_request	= virtblk_init_request,
#ifdef CONFIG_VIRTIO_BLK_SCSI
	.initialize_rq_fn = virtblk_initialize_rq,
#endif
	.map_queues	= virtblk_map_queues,
};
```

blk_mq_dispatch_rq_list-->其中调用queue_rq

virtio_queue_rq-->

virtblk_add_req-->

virtqueue_add_sgs-->

virtqueue_add:向descriptor table添加一个描述符链, 并写入可用环

-->virtqueue_kick_prepare_split: 如果已用环的flags中有VRING_USED_F_NO_NOTIFY, 则表明设备不想要driver发出新的请求, driver则不发.

### device收到请求

对于qemu中的blk设备, 收到请求会首先进入virtio_blk_handle_output-->

virtio_blk_handle_vq-->

--> 从可用环中依次取出描述符链, 并形成IO向量. 

--> 将这些IO向量提交到真实硬件

-->virtio_blk_submit_multireq

submit_requests

### driver收到中断

中断处理函数:

vm_interrupt-->

vring_interrupt-->检查已用环是否变化, 并调用vq的callback

virtblk_done-->

virtqueue_get_buf--->virtqueue_get_buf_ctx--->virtqueue_get_buf_ctx_split

-->detach_buf_split: 

## TODO

- [ ] vhost方案
- [x] rhyper的时候, 数据请求发给root前, 读取desc table这些不会发生冲突, 因为是在同一个CPU上进行的. 
- [ ] 在对used->flags进行操作时, 要对内存进行读写, 而不是cache, 看怎么弄. 可以先不考虑锁的问题, 先让系统跑起来.

先看完qemu的实现方案, 再想想咱们怎么做

两个IO数据请求可能先后到达, EL0怎么先处理完最先的, 再处理之后的??? rhyper也应该存在这个问题......看看异步请求队列

不对, 这个没有关系的, 反正driver发出IO数据请求也是固定的, 对吧...不需要管啦. driver也不会管你发多少次中断给他. 只要能承认中断就行

数据请求通过true和false判断一下就行, 不需要用多少次弄了

配置请求肯定是同步的, 不会说一次好多个, 因此按原来的来就行. 

成功了: 数据请求通过VIRTIO_IO_IN_PROGRESS来判断是否正在进行处理, 非数据请求正常发送. 
