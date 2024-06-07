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

### net driver

* virtio_dev_probe

  net的virtio driver为:

  ```
  static struct virtio_driver virtio_net_driver = {
  	.feature_table = features,
  	.feature_table_size = ARRAY_SIZE(features),
  	.feature_table_legacy = features_legacy,
  	.feature_table_size_legacy = ARRAY_SIZE(features_legacy),
  	.driver.name =	KBUILD_MODNAME,
  	.driver.owner =	THIS_MODULE,
  	.id_table =	id_table,
  	.validate =	virtnet_validate,
  	.probe =	virtnet_probe,
  	.remove =	virtnet_remove,
  	.config_changed = virtnet_config_changed,
  #ifdef CONFIG_PM_SLEEP
  	.freeze =	virtnet_freeze,
  	.restore =	virtnet_restore,
  #endif
  };
  ```

  * virtnet_probe

首先通过alloc_etherdev_mqs函数分配一个net_device. 

```
struct net_device *alloc_etherdev_mqs(int sizeof_priv, unsigned int txqs,
				      unsigned int rxqs)
{
	return alloc_netdev_mqs(sizeof_priv, "eth%d", NET_NAME_UNKNOWN,
				ether_setup, txqs, rxqs);
}
```

设定net_device dev的netdev_ops为virtnet_netdev. ethtool_ops为virtnet_ethtool_ops

init_vqs-->

​	将`refill_work`函数加入到延迟工作队列中, 负责每隔一段时间后向接收队列中加入缓冲区. refill_work调用了try_fill_recv.

​	使用NAPI技术为rq和sq分别设置polling function:

```
netif_napi_add(vi->dev, &vi->rq[i].napi, virtnet_poll,
       napi_weight);
netif_tx_napi_add(vi->dev, &vi->sq[i].napi, virtnet_poll_tx,
      napi_tx ? napi_weight : 0);
```

virtnet_find_vqs-->设置rxq的callback函数为skb_recv_done, txq为skb_xmit_done

## blk-IO请求

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

vring_interrupt-->遍历该设备的所有vqs, 检查每个vq的已用环是否变化, 并调用vq的callback

virtblk_done-->

virtqueue_get_buf--->virtqueue_get_buf_ctx--->virtqueue_get_buf_ctx_split

-->detach_buf_split: 

## net-IO请求

xmit_skb



->virtqueue_add_outbuf

->virtqueue_add

->virtqueue_add_split

当virtio-net driver向rxq中填充空闲buffer后,会notify rxq

### sk_buff

网络数据是以sk_buff结构体在内核中保存的. sk_buff中包含了该网络数据的相关信息, 以及其数据所在的缓冲区地址信息. 如下图:

 <img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240313084719709.png" alt="image-20240313084719709" style="zoom: 33%;" />

刚分配好sk_buff后, head, data, tail是在同一位置. 之后通过变更sk_buff, 来扩展数据区

针对sk_buff的相关操作:

* 分配sk_buff: `alloc_skb`
* 释放sk_buff: `kfree_skb`
* 变更sk_buff
  * skb_put: 在sk_buff的数据区尾部tail后移n个字节, 返回值为拓展出来的那一段数据区的首地址
  * skb_push: 在sk_buff数据区头部data向head方向移动n个字节, 返回值为新数据区首地址
  * skb_pull: 将sk_buff数据区头部data向end方向移动n个字节, 返回值为新数据区首地址
  * skb_reserve: 调整头部大小, 也就是将data和tail同时向end方向移动n个字节, 无返回值



参考资料:

https://www.jianshu.com/p/3c5d5fa339fc

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
