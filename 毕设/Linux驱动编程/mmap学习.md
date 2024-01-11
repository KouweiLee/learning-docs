# mmap学习

## 用户态中的mmap函数

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

参数: 

- `addr`：指定映射的虚拟内存地址，可以设置为 NULL，让 Linux 内核自动选择合适的虚拟内存地址。
- `length`：映射的长度。
- `prot`：映射内存的保护模式，可选值如下：
  - `PROT_EXEC`：可以被执行。
  - `PROT_READ`：可以被读取。
  - `PROT_WRITE`：可以被写入。
  - `PROT_NONE`：不可访问。
- `flags`：指定映射的类型，常用的可选值如下：
  - `MAP_FIXED`：使用指定的起始虚拟内存地址进行映射。
  - `MAP_SHARED`：与其它所有映射到这个文件的进程共享映射空间（可实现共享内存）。
  - `MAP_PRIVATE`：建立一个写时复制（Copy on Write）的私有映射空间。
  - `MAP_LOCKED`：锁定映射区的页面，从而防止页面被交换出内存。

* offset: 要映射文件起始地址的偏移量, 必须按页对齐.  

返回值:

映射到的虚拟地址

## 问题

- [ ] mmap是什么必须按页对齐?


- [x] 如果将RAM内存映射给non root时, root不取消映射, 那么就不需要在el2增加映射了, 直接让用户态能获得el1的ipa就行了. 


