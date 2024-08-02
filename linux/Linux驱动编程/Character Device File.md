# Device File学习

## Device File是什么

Device File是一个表示设备的File,用户态可以通过像读写文件一样来对设备进行操作. 所有的device files都存储在`/dev`目录下. 

有一个特别的device file, `/dev/null`就像一个字节池, 任何写入该设备的操作都会成功,但毫无作用.

## 创建Device file

以下说的Device file,都是char device.

分有3步:

1. include头文件**`linux/device.h`**, **`linux/kdev_t.h`**
2. 创建class

```c
struct class * class_create(struct module *owner, const char *name);
```

> owner – pointer to the module that is to “own” this struct class
> name – pointer to a string for the name of this class

3. 创建Device

```c
struct device *device_create(struct *class, struct device *parent, dev_t dev, void * drvdata, const char *fmt, ...);
```

> class – pointer to the struct class that this device should be registered to
>
> parent – pointer to the parent struct device of this new device, if any
>
> devt – the dev_t for the char device to be added. dev_t本身是32bit数,其中12位为major number, 20位为minor number. 
>
> drvdata – the data to be added to the device for callbacks
>
> fmt – string for the device’s name
>
> ... – variable arguments

### 示例

```c
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kdev_t.h>
#include <linux/fs.h>
#include <linux/err.h>
#include <linux/device.h>
 
dev_t dev = 0;
static struct class *dev_class;

static int __init hello_world_init(void)
{
        /*Allocating Major number*/
        if((alloc_chrdev_region(&dev, 0, 1, "etx_Dev")) <0){
                pr_err("Cannot allocate major number for device\n");
                return -1;
        }
        pr_info("Major = %d Minor = %d \n",MAJOR(dev), MINOR(dev));
 
        /*Creating struct class*/
        dev_class = class_create(THIS_MODULE,"etx_class");
        if(IS_ERR(dev_class)){
            pr_err("Cannot create the struct class for device\n");
            goto r_class;
        }
 
        /*Creating device*/
        if(IS_ERR(device_create(dev_class,NULL,dev,NULL,"etx_device"))){
            pr_err("Cannot create the Device\n");
            goto r_device;
        }
        pr_info("Kernel Module Inserted Successfully...\n");
        return 0;
 
r_device:
        class_destroy(dev_class);
r_class:
        unregister_chrdev_region(dev,1);
        return -1;
}
```

## 对device file的IO操作

如果希望对device file能进行读写操作,则需要在driver中注册.

### cdev struct

Linux内核中表示文件的数据结构为inode. 当inode表示一个char device file时,其中的cdev域会指向一个cdev结构体, 这个结构体用来表示char device. 该结构体的定义如下:

```c
struct cdev { 
    struct kobject kobj; 
    struct module *owner; 
    const struct file_operations *ops; 
    struct list_head list; 
    dev_t dev; 
    unsigned int count; 
};
```

其中需要我们初始化的两个域为owner和ops. 

创建一个cdev有两种方式:

1. 动态分配

```c
struct cdev *my_cdev = cdev_alloc( );
my_cdev->ops = &my_fops;
```

2. 拥有者分配

```c
void cdev_init(struct cdev *cdev, struct file_operations *fops);
```

创建好cdev后, 可通过下面的函数激活这个device:

```c
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
```

> **`num `**is the first device number to which this device responds, and
>
> **`count `**is the number of device numbers that should be associated with the device.

device激活后,就可以对其进行文件操作了.

### file operations

file operations总共支持这么多操作, 如果操作为NULL, 表示该操作不支持. 第一个参数通常为**`THIS_MODULE`**, 定义在`<linux/module.h>`, 用于当卸载module时检查是否有正在进行的ops. 

```c
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *); // device -> user
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *); // user -> device
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
    int (*iterate) (struct file *, struct dir_context *);
    int (*iterate_shared) (struct file *, struct dir_context *);
    unsigned int (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long); // 当内核和用户均为64位时,是这个ioctl
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);// 当内核为64位,用户程序为32位时,是这个ioctl
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *); // open, 可以设置为NULL
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *); // 当file结构体被释放时,即用户态调用close函数时,会调用该函数,可以为NULL
    int (*fsync) (struct file *, loff_t, loff_t, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
    ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    int (*check_flags)(int);
    int (*flock) (struct file *, int, struct file_lock *);
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
    int (*setlease)(struct file *, long, struct file_lock **, void **);
    long (*fallocate)(struct file *file, int mode, loff_t offset,
              loff_t len);
    void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
    unsigned (*mmap_capabilities)(struct file *);
#endif
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
            loff_t, size_t, unsigned int);
    int (*clone_file_range)(struct file *, loff_t, struct file *, loff_t,
            u64);
    ssize_t (*dedupe_file_range)(struct file *, u64, u64, struct file *,
            u64);
};
```

### 有用的接口函数

内核中,有些有用的接口函数,被用在file operations中. 他们分别是:

#### kmalloc

```c
#include <linux/slab.h>
void *kmalloc(size_t size, gfp_t flags);
```

类似于用户态的malloc函数,该函数分配size字节的内存区域(该区域物理地址连续), flags表示分配内存的类型, 可以为:

> **`GFP_USER`** – Allocate memory on behalf of the user. May sleep.
>
> **`GFP_KERNEL`** – Allocate normal kernel ram. May sleep.
>
> **`GFP_ATOMIC`** – Allocation will not sleep. May use emergency pools. For example, use this inside interrupt handler.
>
> **`GFP_HIGHUSER`** – Allocate pages from high memory.
>
> **`GFP_NOIO`** – Do not do any I/O at all while trying to get memory.
>
> **`GFP_NOFS`** – Do not make any fs calls while trying to get memory.
>
> **`GFP_NOWAIT`** – Allocation will not sleep.
>
> **`__GFP_THISNODE`** – Allocate node-local memory only.
>
> **`GFP_DMA`** – Allocation is suitable for DMA. Should only be used for **`kmalloc`** caches. Otherwise, use a slab created with SLAB_DMA.
>
> Also, it is possible to set different flags by OR’ing in one or more of the following additional ***`flags`\***:
>
> **`__GFP_COLD`** – Request cache-cold pages instead of trying to return cache-warm pages.
>
> **`__GFP_HIGH`** – This allocation has high priority and may use emergency pools.
>
> **`__GFP_NOFAIL`** – Indicate that this allocation is in no way allowed to fail (think twice before using).
>
> **`__GFP_NORETRY`** – If memory is not immediately available, then give up at once.
>
> **`__GFP_NOWARN`** – If allocation fails, don’t issue any warnings.
>
> **`__GFP_REPEAT`** – If allocation fails initially, try once more before failing.

#### kfree

```c
void kfree(const void *objp)
```

类似于用户态的free函数. objp指向由kmalloc返回的指针.

#### copy_from_user

```c
unsigned long copy_from_user(void *to, const void __user *from, unsigned long  n);
```

从user space拷贝n字节的数据到kernel space, 返回值为不能被拷贝的字节数, 应该为0.

#### copy_to_user

```c
unsigned long copy_to_user(const void __user *to, const void *from, unsigned long  n);
```

从kernel space拷贝n字节的数据到user space, 返回值为不能被拷贝的字节数, 应该为0.

## 参考资料

1. https://embetronicx.com/tutorials/linux/device-drivers/device-file-creation-for-character-drivers/
2. https://embetronicx.com/tutorials/linux/device-drivers/cdev-structure-and-file-operations-of-character-drivers/
3. https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-programming/