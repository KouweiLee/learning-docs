# Linux开发文档

## 碎杂知识

__user：告诉编译器这是一个用户态的地址，在内核态时不要解引用。

## 内存管理

copy_from_user：从用户空间拷贝数据到内核空间。

```c
unsigned long copy_from_user (void * to, const void __user * from, unsigned long n);
```

* 在内核中动态申请空间

kmalloc：申请一片较小的内存区域，物理地址是连续的，其虚拟地址和物理地址之间的转换仅仅是一个偏移量，通过__pa可以获得其物理地址

vmalloc：类似于正常的内存分配函数

## 进程管理

mutex_lock_interruptible函数：这个锁是可中断的，当等待锁时，如果收到信号，则会退出并返回一个错误码。
