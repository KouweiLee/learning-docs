# sysfs学习

sysfs和IOCTL类似，都是用于内核空间与用户空间的通信。sysfs是内核暴露的一个虚拟文件系统，被挂载在`/sys`下，其中的文件包含设备和驱动的信息，也可以对设备进行控制。通常用来暴露内核中某些设备的信息给用户空间。

## kernel object

sysfs的核心是kernel object，Kobject是连接sysfs和kernel的桥梁，由`struct kobject`表示：

```c
<linux/kobject.h>
struct kobject {
    char *k_name;
    char name[KOBJ_NAME_LEN];
    struct kref kref;
    struct list_head entry;
    struct kobject *parent;
    struct kset *kset;
    struct kobj_type *ktype;
    struct dentry *dentry;
};
```

一个kobject就对应一个`/sys`下的目录，`parent`则指定了目录之间的层级关系。换言之，kobject最大的作用就是用来创建`/sys`下的目录。

## 使用sysfs

使用sysfs分为两步：

1. **在`/sys`下创建目录**

通过下面的函数即可在`/sys`下创建目录：

```c
struct kobject * kobject_create_and_add ( const char * name, struct kobject * parent);
```

* name：kobject的名字
* parent：父目录对应的kobject

如果kobject没用了需要释放，则调用`kobject_put`

2. **创建sysfs文件**

