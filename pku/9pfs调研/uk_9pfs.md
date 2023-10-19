# unikraft 9pfs

在一个计算机系统中，可以同时包含多个持久存储设备，它们上面的数据可能是以不同文件系统格式存储的。为了能够对它们进行统一管理，在内核中有一层 **虚拟文件系统** (VFS, Virtual File System) ，它规定了逻辑上目录树结构的通用格式及相关操作的抽象接口，只要不同的底层文件系统均实现虚拟文件系统要求的那些抽象接口，再加上 **挂载** (Mount) 等方式，这些持久存储设备上的不同文件系统便可以用一个统一的逻辑目录树结构一并进行管理。

```c
#define UK_FS_REGISTER(fssw) static struct vfscore_fs_type	\
	__attribute((__section__(".uk_fs_list")))		\
	*__ptr_##fssw __used = &fssw;	

UK_FS_REGISTER(uk_9pfs_fs);

static struct vfscore_fs_type uk_9pfs_fs = {
	.vs_name	= "9pfs",
	.vs_init	= NULL,
	.vs_op		= &uk_9pfs_vfsops
};

struct vfsops uk_9pfs_vfsops = {
	.vfs_mount	= uk_9pfs_mount,
	.vfs_unmount	= uk_9pfs_unmount,
	.vfs_sync	= uk_9pfs_sync,
	.vfs_vget	= uk_9pfs_vget,
	.vfs_statfs	= uk_9pfs_statfs,
	.vfs_vnops	= &uk_9pfs_vnops
};
```

## 以read为例



```c
struct vfscore_file {
	int fd;
	int		f_flags;	/* open flags */
	int		f_count;	/* reference count */
	off_t		f_offset;	/* current position in file */
	void		*f_data;        /* file descriptor specific data */
	int		f_vfs_flags;    /* internal implementation flags */
	struct dentry   *f_dentry;
	struct uk_mutex f_lock;

	struct uk_list_head f_ep;	/* List of eventpoll_fd's */
};
```



```c
struct uio {
	struct iovec *uio_iov;		/* scatter/gather list */
	int	uio_iovcnt;		/* length of scatter/gather list */
	off_t	uio_offset;		/* offset in target object */
	ssize_t	uio_resid;		/* remaining bytes to process */
	enum	uio_rw uio_rw;		/* operation */
};
```



```
	error = VOP_READ(vp, fp, uio, 0);

```

uk_9pfs_read(9pfs) -> uk_9p_read(uk9p)

在uk_9p_read中则是使用了plan 9协议，将请求头序列化进uk_9preq中，然后通过send_and_wait_zc函数发送请求。最后调用dev->ops->request(dev, req)，执行plat/drivers/virtio/virtio_9p.c中的`virtio_9p_request`函数。



virtqueue中notify函数的具体实现

```c
static int vpci_legacy_notify(struct virtio_dev *vdev, __u16 queue_id)
{
	struct virtio_pci_dev *vpdev;

	UK_ASSERT(vdev);
	vpdev = to_virtiopcidev(vdev);
	virtio_cwrite16((void *)(unsigned long) vpdev->pci_base_addr,
			VIRTIO_PCI_QUEUE_NOTIFY, queue_id);

	return 0;
}
```

## 9p 协议

[9p相关基础知识 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/9p基础知识/)

[intro(9P) - Plan 9 from User Space (9fans.github.io)](https://9fans.github.io/plan9port/man/man9/intro.html)

9p，全称The Plan 9 File Protocol，被用来规范9p client和 server之间的消息通讯。

9p server提供分层文件系统（文件树）的代理，并响应client发出的创建、删除、读取、写入文件的请求。

消息通信的流程是，client向server发送requests(T-message)，随后server向client返回replies(R-message)。

消息的格式为：消息头部是固定的7个字节，前4个字节表示整个消息的长度。第5个字节表示消息的类型（在request_create时设定）。后两个字节表示消息的标签tag，用来验证，R-message和T-message中tag应该相同。之后的字节序列时根据不同类型的消息不定长的，

属于消息的数据内容，它前两个字节表示数据的长度n，之后的n个字节均是数据。字符串用UTF-8编码，不以空字符结尾，空字符在9p中是违法的。

例如：

```
size[4] Tversion tag[2] msize[4] version[s]
size[4] Tattach tag[2] fid[4] afid[4] uname[s] aname[s]
size[4] Rattach tag[2] qid[13]
```

说明Tversion信息的前7个字节就是上面固定的，之后用4个字节表示消息的最大长度msize，s个字节表示version，version的前2个字节表示后面数据的长度。

* 连接的建立过程

> 一些注意事项：
>
> 每个T-message和R-message都有一个tag域，它们应该相同，并由client进行分配。在同一次连接中不同消息的tag应保证不同。
>
> 大多数T-message包含一个fid，由client用来标志server上的一个文件。fid类似于文件描述符，由client指定，且同一个连接共享相同的fid空间。

1. client发送version message，和server确认协议版本和消息最大长度，并初始化连接。version message的tag为NOTAG。
2. client发送attach message。该message中，fid会被作为服务端文件树根目录的fid。server返回的attach message中，qid是server上被访问文件的独一标识，可能有多个fid指向server是哪个同一个文件，但只有一个qid对于这个文件。
3. 

## vfscore

VFS：虚拟文件系统，是具体文件系统和上层应用的接口层。9pfs就是一种具体文件系统。

![image-20230905151628634](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309051516731.png)



## vfscore核心数据结构

[vfscore核心数据结构 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/23/vfscore核心数据结构/)

`vfscore` (*Virtual File System Core*) ，提供了对文件系统管理的通用系统调用接口。有四个非常重要的数据结构：vfscore_file，dentry，mount，vnode

### vfscore_file

文件对象结构体，VFS对一个文件的描述。文件对象是在文件被打开时创建的。

```c
struct vfscore_file {
	int fd;  /* 文件描述符，用于让进程快速索引到文件位置 */
	int		f_flags;	/* open flags  打开标志 */
	int		f_count;	/* reference count  引用计数 */
	off_t		f_offset;	/* current position in file  文件指针，用于定位当前指针在文件中的哪个位置 */
    // 对于9p，则指向uk_9pfs_file_data
	void		*f_data;       
	int		f_vfs_flags;    /* internal implementation flags 实现标志（可能是区分不同类型文件的标识符） */
	struct dentry   *f_dentry;  /* 该文件所在的目录项 */
	struct uk_mutex f_lock;  /* 文件锁，用于同步操作 */

	struct uk_list_head f_ep;	/* List of eventpoll_fd's eventpoll的文件描述符的链表 */
};
```

### dentry

表示目录项结构体。VFS把每个目录看作一个文件，由若干子目录和文件组成。目录项对象将每个分量与其对应的索引节点相联系。例如/tmp/test，内核会分别为“/”，“tmp”，“test”创建目录项。目录项也可以用于标识普通文件。

```c
struct dentry {
	struct uk_hlist_node d_link;	/* link for hash list  */
	int		d_refcnt;	/* reference count  引用计数 */
	char		*d_path;	/* pointer to path in fs dentry所代表的相对mount的相对路径 */
    // 该dentry对应的vnode，描述该目录项所代表文件的各种属性
	struct vnode	*d_vnode; 
	struct mount	*d_mount;  /* 该目录项所指向的挂载点 */
	struct dentry   *d_parent; /* pointer to parent 指向父目录的指针*/
	struct uk_list_head d_names_link; /* link fo vnode::d_names 详见2.4*/
	struct uk_mutex	d_lock; /* 目录项锁，用于同步操作 */
	struct uk_list_head d_child_list;
	struct uk_list_head d_child_link;  /* 详见2.5 */
};
```

* `struct uk_hlist_node d_link`

dentry会被记录到一个哈希表中，该哈希表的定义为：

```c
static struct uk_hlist_head dentry_hash_table[DENTRY_BUCKETS];
```

其中`uk_hlist_head `是一个链表头指针，该Hash表因此被表示一个链表的数组。通过计算dentry中的路径和其指向mount节点的信息，计算出Hash值k，则会插入到`dentry_hash_table[k]`链表中。

### mount

挂载结构体。挂载的作用，就是将一个设备（存储设备）挂接到一个已知目录下，访问该目录就是访问该存储设备。

linux操作系统将所有的设备都看作文件，它将整个计算机的资源都整合成一个大的文件目录。我们要访问存储设备中的文件，必须将文件所在的分区挂载到一个已存在的目录上，然后通过访问这个目录来访问存储设备。挂载就是把设备放在一个目录下，让系统知道怎么管理这个设备里的文件，了解这个存储设备的可读写特性之类的过程。

```c
struct mount {
	struct vfsops	*m_op;		/* pointer to vfs operation 详见3.1 */
	int		m_flags;	/* mount flag  挂载标志 */
	int		m_count;	/* reference count  引用计数 */
	char            m_path[PATH_MAX]; /* mounted path 详见3.2 */
	char            m_special[PATH_MAX]; /* resource 该mount所挂载的设备在文件系统中的绝对路径。*/
	struct device	*m_dev;		/* mounted device 指向mount所挂载的设备 */
	struct dentry	*m_root;	/* root vnode */
	struct dentry	*m_covered;	/* 指向挂载路径对应的dentry */
	void		*m_data;	/* private data for fs 文件系统的私有数据 */
    // 所有mount实例都会通过该mnt_list链接成一个环
	struct uk_list_head mnt_list;
	fsid_t 		m_fsid; 	/* id that uniquely identifies the fs 该文件系统的id */
};
```

* `struct vfsops	*m_op`

通常来讲，一个mount挂载设备对应的是一个文件系统（如ext2，9p等），而m_op指向的是该类型文件系统的基本操作，只要具体的文件系统实现这些接口，就可以被注册到VFS中统一管理和使用。vfsops结构体的定义如下：

```c
struct vfsops {
	int (*vfs_mount)	(struct mount *, const char *, int, const void *);
	int (*vfs_unmount)	(struct mount *, int flags);
	int (*vfs_sync)		(struct mount *);
	int (*vfs_vget)		(struct mount *, struct vnode *);
	int (*vfs_statfs)	(struct mount *, struct statfs *);
	struct vnops	*vfs_vnops;
};
```

* `struct dentry *m_root;` 与 `struct dentry *m_covered;`

如果文件系统挂载到`/home/user/fs0`下，则`m_covered`指向该路径对应的dentry。而`m_root`指向的是，挂载后所创建的root dentry。

![image-20230905191024755](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309051910899.png)

### vnode

文件信息管理结构体，用于管理文件的具体信息：

```c
struct vnode {
	uint64_t	v_ino;		/* inode number 该vnode所关联的底层inode的编号 */
	struct uk_list_head v_link;	/* link for hash list  详见4.1 */
	struct mount	*v_mount;	/* mounted vfs pointer 该vnode所指向的挂载点*/
	struct vnops	*v_op;		/* vnode operations 详见4.2 */
	int		v_refcnt;	/* reference count  vnode引用计数 */
	int		v_type;		/* vnode type 表示该文件的具体类型 */
	int		v_flags;	/* vnode flag 文件访问的模式 */
	mode_t		v_mode;		/* file mode 文件的权限控制 */
	off_t		v_size;		/* file size  文件大小 */
	struct uk_mutex	v_lock;		/* lock for this vnode  vnode锁，用于同步操作 */
	struct uk_list_head v_names;	/* directory entries pointing at this */
    // 对于9pfs，则是uk_9pfs_node_data
	void		*v_data;	
};
```

* `struct uk_list_head v_link`

所有被open的vnode都会被写入到哈希表`vnode_table`中，类似于dentry。

* `struct vnops *v_op`

表示该vnode支持的操作，例如open、close等。使用这些操作时，要使用对外封装的接口，例如：

```c
#define VOP_OPEN(VP, FP)	   ((VP)->v_op->vop_open)(FP) // 打开文件
```

## 9pfs相关结构

![image-20230905152938995](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202309051529143.png)

### 基础数据结构

#### 9pdev

9pdev是一个专门用来与9p设备进行通信的结构体。

```c
struct uk_9pdev {
	//一些传输操作，包括connect、disconnect、request
	const struct uk_9pdev_trans_ops *ops;
	//用于connect的内存分配器
	struct uk_alloc                 *a; 
	// 设备的状态，包括已连接（默认）和disconnect后的状态
	enum uk_9pdev_trans_state       state;
	// 一个消息的最大大小，可以自己指定
	uint32_t                        msize;
	// 一个消息对于传输的最大大小，这个值是这个dev支持的最大大小
	uint32_t                        max_msize;
	// 要传输的数据
	void                            *priv;
	/* @internal Fid management. */
	struct uk_9pdev_fid_mgmt	_fid_mgmt;
	/* @internal Request management. */
	struct uk_9pdev_req_mgmt        _req_mgmt;
};
```

#### 9pdev_trans

它用来描述传输的结构。

```c
struct uk_9pdev_trans {
	// 传输的名字，例如virtio或者xen
	const char  *name;
	/* Supported operations. */
	const struct uk_9pdev_trans_ops         *ops;
	/* Allocator used for devices which use this transport layer. */
	struct uk_alloc                         *a;
	/* @internal Entry in the list of available transports. */
	struct uk_list_head                     _list;
};
```

#### 9pfs_mount_data

```c
// 用来与上层vfs mount绑定的数据，即mount->m_data = uk_9pfs_mount_data
struct uk_9pfs_mount_data {
	/* 9P device. */
	struct uk_9pdev		*dev;
	/* Wanted transport. */
	struct uk_9pdev_trans	*trans;
	// 9p具体的协议名，现在unikraft仅实现了9P2000.u
	enum uk_9pfs_proto	proto;
	// 尝试在远程服务器上挂载的用户名
	const char		*uname;
	// 当有多个文件系统的时候选择的文件树
	const char		*aname;
};
```

#### 9pfs_file_data

对应一个9p文件。对于VFS，有`vfscore_file->f_data = uk_9pfs_file_data`

```c
struct uk_9pfs_file_data {
	// 和9p文件系统关联的标识符
	struct uk_9pfid        *fid;
	/*
	 * Buffer for persisting results from a 9P read operation across
	 * readdir() calls.
	 */
    // 通过readdir()操作获取的9p读取操作的持久化缓冲区
	char                   *readdir_buf;
	/*
	 * Offset within the buffer where the stat of the next child can
	 * be found.
	 */
	int                    readdir_off;
	// 缓冲区中数据的大小
	int                    readdir_sz;
};
```

#### 9pfs_node_data

对应上层vfs中vnode的结构体，即`vnode->v_data = uk_9pfs_node_data`

```c
struct uk_9pfs_node_data {
	// 与vnode关联的fid
	struct uk_9pfid        *fid;
	// 从这个vnode同时打开的文件数量
	int                    nb_open_files;
	/* Is a 9P remove call required when nb_open_files reaches 0? */
	bool                   removed;
};
```

### 9p协议规定的类型

* 9p type

9p message的类型。名为uk_9p_type，例如建立连接、验证信息、打开文件等等。

* qid type

表示文件的类型

* open mode

文件打开方式

* 9p_qid

共13个字节，回应`auth`, `attach`, `walk`, `open`,  `create`请求的R-message都会带一个qid给client。它表示server对正在访问的文件的唯一标识。

```c
struct uk_9p_qid {
	uint8_t                 type;		// 1字节  即1.6.2中的qid类型，代表文件的类型
	uint32_t                version;	// 4字节  代指文件的版本，即每次修改版本都会增加
	uint64_t                path;		// 8字节  在hierarchy中文件的path是唯一的
};
```

* `9p_fid`

描述一个被管理的fid。

```c
// 描述一个被管理的fid
struct uk_9pfid {
	uint32_t                fid;	// fid号码，是唯一的标识
	struct uk_9p_qid        qid;	// 与fid关联的服务端的qid，代表客户端的一个文件
	uint32_t                iounit;	// 保证原子传输的最大尺寸，因为9P的实现可能会限制单个消息的大小，所以一次读或写调用可能会产生一个以上的消息。
	bool was_removed;	// 代表是否被移除了
	__atomic                refcount;	// 引用计数，即代表有多少在用这个文件
	struct uk_9pdev         *_dev;		// 关联的9p device
	struct uk_list_head     _list;	// 这个fid所在的list，用于9pdev_fid_mgmt来管理fid，即分配fid
};
```

* `9p_str`
* `uk_9p_stat`

描述文件的元数据。

### 9preq_fcall

```c
struct uk_9preq_fcall {
	// 整个fcall的大小，最初是buffer的大小，在传输和接受就绪后将变成发送/接受的消息的大小。
	uint32_t                        size;
	/* Type of the fcall. Should be T* for transmit, R* for receive. */
	uint8_t                         type;
	/* Offset while serializing or deserializing. */
	uint32_t                        offset;
	// 指向9preq中的xmit_buf（发送）/recv_buf（接受）
	void                            *buf;

	/* Zero-copy buffer pointer. */
	void                            *zc_buf;
	/* Zero-copy buffer size. */
	uint32_t                        zc_size;
	/* Zero-copy buffer offset in the 9P message. */
	uint32_t                        zc_offset;
};
```

### 9preq

该结构体描述一个9P request，通过uk_9pdev_req_create()来创建。

```c
struct uk_9preq {
	/*
	 * Fixed-size buffer for transmit, used for most messages.
	 * Large messages will always zero-copy from the user-provided
	 * buffer (on Twrite).
	 */
    // 发送缓冲区
	uint8_t				xmit_buf[UK_9P_BUFSIZE];
	/*
	 * Fixed-size buffer for receive, used for most messages.
	 * Large messages will always zero-copy into the user-provided
	 * buffer (on Tread).
	 */
    // 接收缓冲区
	uint8_t				recv_buf[UK_9P_BUFSIZE];
	/* 2 KB offset in the structure here. */
    
	// 用于发送的数据结构
	struct uk_9preq_fcall           xmit;
	// 用于接收的数据结构
	struct uk_9preq_fcall           recv;
	/* State of the request. See the state enum for details. */
	enum uk_9preq_state             state;
	// 9p req的tag，用于验证消息
	uint16_t                        tag;
	/* Entry into the list of requests (API-internal). */
	struct uk_list_head             _list;
	/* @internal 9P device this request belongs to. */
	struct uk_9pdev                 *_dev;
	/* @internal Allocator used to allocate this request. */
	struct uk_alloc                 *_a;
	/* Tracks the number of references to this structure. */
	__atomic                        refcount;
};
```

其中state，代表request的状态，分为五种：NONE（刚刚被创建）、INITIALIZED（初始化了——可以开始接受序列化的数据），READY（准备好了——可以发送了），SENT（已经发送到传输层了）、RECEIVED（从传输层接收到信息并且重要的9P信息例如类型、标签、大小都已经被验证了）。

### 如何生成9preq

创建一个与9pdev绑定的9preq，是在函数`uk_9pdev_req_create`中，type为9p 消息的类型9p_type。

```c
struct uk_9preq *uk_9pdev_req_create(struct uk_9pdev *dev, uint8_t type)
```

该函数调用`uk_9preq_init`，初始化9preq。

当要从9preq缓冲区读或写数据时，调用函数：

```c
static inline int uk_9preq_writebuf(struct uk_9preq *req, const void *buf, uint32_t size)
static inline int uk_9preq_readbuf(struct uk_9preq *req, void *buf, uint32_t size)
```

### 与9pdev相关的基础操作

首先提到一个注册函数，该函数将一个`9pdev_trans`结构体注册到`uk_9pdev_trans_saved_default`中

```c
static struct uk_9pdev_trans *uk_9pdev_trans_saved_default; // 一个默认的trans

int uk_9pdev_trans_register(struct uk_9pdev_trans *trans)
```

创建9pdev的函数，该函数根据trans创建9pdev，并调用trans的connect函数，连接到virtio_9p_dev设备：

```c
struct uk_9pdev *uk_9pdev_connect(const struct uk_9pdev_trans *trans,
				const char *device_identifier,
				const char *mount_args,
				struct uk_alloc *a)
```

### 9pfs的操作

挂载9p文件系统的操作：

```c
static int uk_9pfs_mount(struct mount *mp, const char *dev,
			int flags __unused, const void *data)
```

挂载流程：

1. 执行`uk_9pdev_connect`函数，

```c
static const struct uk_9pdev_trans_ops v9p_trans_ops = {
	.connect        = virtio_9p_connect,
	.disconnect     = virtio_9p_disconnect,
	.request        = virtio_9p_request // 上层调用的request函数
};

```

## 9pfs在unikraft中分层结构

共分为5层：

```
c语言标准库函数
---------------
vfscore
---------------
9pfs
---------------
uk9p
---------------
plat/virtio
```



### 9pfs

接口函数：

1. 挂载9pfs：`uk_9pfs_mount`

```c
// 会注册为上层的vfsops.vfs_mount
static int uk_9pfs_mount(struct mount *mp, const char *dev, 
int flags __unused, const void *data)
```

调用`uk_9pdev_connect`函数，且执行`mount_data.dev <- 9pdev`。

调用uk_9p_version，与9p server建立一个会话，9p会话的第一个消息就是协商version。

### uk9p

接口函数:

1. 建立和virtio中dev的连接：`uk_9pdev_connect`

```c
struct uk_9pdev *uk_9pdev_connect(const struct uk_9pdev_trans *trans,
const char *device_identifier,
const char *mount_args,
struct uk_alloc *a)
```

创建一个9pdev，并且这个dev拥有trans相关的操作，并通过调用virtio_9p_connect函数与底层virtio_9p_dev进行连接。

2. 与9p server协商version：` uk_9p_version`
3. 发送请求函数：`send_and_wait_no_zc`

该函数通过`uk_9pdev_request`向virtio层发送request，之后调用`uk_9preq_waitreply`**等待**qemu的中断通知。

当qemu执行完req后，会通过中断来通知，对于unikraft，中断的处理函数是virtio_pci_handle

### virtio层

接口函数：

1. 建立和9pfs中dev的连接：`virtio_9p_connect`

```c
// 将device_identifier对应的virtio_9p_device和uk_9pdev建立连接
static int virtio_9p_connect(struct uk_9pdev *p9dev,
			     const char *device_identifier,
			     const char *mount_args __unused)
```

2. 发送request：virtio_9p_request`	

依次调用了：virtqueue_buffer_enqueue

virtqueue_host_notify->vpci_legacy_notify

该函数*向virtio-9pfs设备对应的pci总线发送通知，指示queue_id有数据*

当qemu完成操作后，会通过中断来通知unikraft。

3. 中断处理函数：`virtio_pci_handle`，当qemu发出pci中断时，unikraft会执行该函数。该函数会最终执行vq的callback函数，实际上就是`virtio_9p_recv`

## 参考资料

下面的很多在这个文件夹下：[unikraft文件系统](https://tenonos.github.io/categories/#collapse%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E4%B8%8E%E5%AD%98%E5%82%A8)

- [x] [vfscore核心数据结构 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/23/vfscore核心数据结构/)
- [x] [9p相关基础知识 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/9p基础知识/)
- [x] [unikraft文件系统导览 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/unikraft文件系统导览/)
- [ ] 最重要的：[9pfs相关结构以及源码分析 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/10/30/9pfs/#2-1-9preq-init-9preq-c)
- [ ] [9pfs kvm qemu源码流程 | TenonOS-unikraft-learning](https://tenonos.github.io/2022/11/07/9pfs kvm qemu源码流程/#unikraft侧)：讲了*当qemu完成9p操作后，如何发通知给unikraft的*

virtio相关：

- [x] [网络虚拟化 virtio 框架分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/542483879)

unikraft相关：

- [ ] [Setup – Unikraft](https://unikraft.org/community/hackathons/sessions/setup/)：讲了几个例子，如何启动Unikraft，session 2讲了如何使用9pfs

## 问题

- [x] fid的作用是什么
- [x] 涉及9pfs的各个文件夹都有什么作用，都属于哪个层次
- [x] ```c
  struct virtio_9p_device {
  	/* Virtio device. */
  	struct virtio_dev *vdev;
  	/* Mount tag for this virtio device. */
  	char *tag;
  	/* Entry within the virtio devices' list. */
  	struct uk_list_head _list;
  	/* Virtqueue reference. */
  	struct virtqueue *vq;
  	/* Hw queue identifier. */
  	uint16_t hwvq_id;
  	/* libuk9p associated device (NULL if the device is not in use). */
  	struct uk_9pdev *p9dev;
  	/* Scatter-gather list. */
  	struct uk_sglist sg;
  	struct uk_sglist_seg sgsegs[NUM_SEGMENTS];
  	/* Spinlock protecting the sg list and the vq. */
  	__spinlock spinlock;
  };
  ```

  这其中的mount tag是什么意思?
  
  qemu会将shared path用mount tag表示，guest通过mount tag来识别qemu共享的文件。

