# ArceOS整体架构

## crates

[Index of crates (rcore-os.cn)](http://rcore-os.cn/arceos/)

Crates为Modules实现提供更底层的支持，通用的基础库，与os无关。

### allocator

[allocator - Rust (rcore-os.cn)](http://rcore-os.cn/arceos/allocator/index.html)

提供了3种分配算法，使用统一的接口：

bitmap：位图，每一位表示一页是否被占用。面向页的分配

buddy：伙伴系统中的内存分配，面向字节的分配

slab：基于slab分配算法的字节分配

### arm_gic

[arm_gic - Rust (rcore-os.cn)](http://rcore-os.cn/arceos/arm_gic/index.html)

ARM GIC寄存器的定义以及基本操作。

### arm_pl011

针对PL011 UART的类型定义

### axerrno

[axerrno - Rust (rcore-os.cn)](http://rcore-os.cn/arceos/axerrno/index.html)

ArceOS的错误码定义

### axfs_devfs

ArceOS的设备文件系统	

### axfs_ramfs

ArceOS的RAM文件系统。



### axfs_vfs

ArceOS的文件系统接口。

### axio

类似于std::io，提供读、写、定位等操作

### capability

提供权限的定义，包括读、写、执行。

### crate_interface

通常用于解决crates之间的循环依赖。

### driver_block

块设备驱动程序(即磁盘)的常见特征和类型。包括读一个块、写一个块

### driver_common

设备驱动的通用接口，需要结合具体设备的crate使用。

### driver_display

图像显示设备驱动的常见特征和类型。

### driver_net

网络设备的常见特征和类型

### driver_pci

PCI总线操作的数据结构与函数

### driver_virtio

Virtio设备相关的一些东西吧。

### flatten_objects

是一个容器，可以用于存储编号对象，也就是每个在该容器中的对象，都会有一个独一的编号，利用该编号可以从容器中取出对象。

### handler_table

一个无锁的event处理的handler

### kernel_guard

创建一个临界区，禁止本地中断或抢占。用来在内核中实现一个自旋锁。

> 自旋锁（Spin Locks）：一个线程无法获取自旋锁时，不会阻塞，而是会循环等待。

### lazy_init

一个可延迟初始化的值的包装器。未提供并发的安全保证，只能被初始化一次。

### linked_list

链表。

### memory_addr

物理地址和虚拟地址的包装器，并提供一些拓展功能，比如获取指定align后的地址

### page_table

提供通用、面向不同架构的页表数据结构。支持x86、ARM和RISC-V。

### page_table_entry

面向不同硬件架构的页表项的定义。

### percpu

定义和获取per-CPU的数据结构。

> 每个CPU都有它自己的per-CPU数据区域，可以通过per-CPU数据区域实现对CPU的读操作。

### percpu_macros

对per-CPU数据结构定义和访问的宏，注意不要直接使用该crate，使用percpu。

### ratio

包装一个分数，并提供了相关操作。

### scheduler

一个统一的接口，支持不同调度算法。

目前支持的调度算法有：

1. FIFO
2. RRS：循环调度程序，抢占式
3. CFS：抢占式，完全公平调度程序

### slab_allocator

slab的分配器

### spinlock

自旋锁，拿到锁的线程会一直保持该锁，直到显式释放锁；没有拿到锁的一直忙等。

### timer_list

一个计时事件的列表，当计时器超时后会逐个触发。

### tuple_for_each

提供宏和方法，来对元组结构体的每个元素进行迭代。

## modules

### axalloc

全局内存分配器

### axconfig

对应平台的常数和参数。

### axdisplay

图形模块

### axdriver

设备驱动模块

### axfs

文件系统模块。为不同的文件系统提供统一的文件系统操作，

### axhal

硬件抽象层，为不同平台定义了引导指令和初始化流程，并提供了针对硬件的操作。

虚拟地址到物理地址的映射是一个线性映射，就是-0

* memory_regions函数



### axlog

多级日志模块，从低到高依次为：error, warn, info, debug, trace。

设置了最大日志等级时，包含该等级以及低等级的日志会被打印。

### axnet

为TCP/UDP提供统一的网络原语

### axruntime

任何app都应该链接到该模块，该模块在进入app的main函数前进行一些初始化工作。

其中rust_main和rust_main_secondary函数为ArceOS runtime的入口点和辅助CPU的入口点。

Features均默认禁用。

### axsync

提供同步互斥原语，包括：

`Mutex`：类似于`std::sync::Mutex`的互斥锁

`spin`：自旋锁

### axtask

任务调度管理模块，包括任务创建、调度、休眠、销毁等

## axlibc

### assert.c

提供了断言失败函数，`_Noreturn`表示该函数不会返回到调用它的地方

### ctype.c

tolower和toupper函数

### dirent.c

未实现

### dlfcn.c

未实现

### env.c

未实现

### epoll.c

用于在 Linux 操作系统上使用 epoll I/O 事件通知机制

### fcntl.c

未实现

### flock.c

未实现

`flock` 函数是用于文件锁定（file locking）的 POSIX 标准函数。

### fnmatch.c

[musl/src/regex/fnmatch.c at kraj/master · kraj/musl (github.com)](https://github.com/kraj/musl/blob/kraj/master/src/regex/fnmatch.c)

模式匹配，这个好实现。

- [x] todo

### glob.c

用于文件系统中文件名的模式匹配

### ioctl.c

IO控制函数

### libgen.c

获取文件路径字符串中的目录名和文件名

### libm.c

数学函数

### locale.c

设置程序的本地化环境，例如日期、时间等等

未实现，感觉还能实现

- [ ] todo

### log.c

实现了log函数

### math.c

实现了许多math函数，也有许多未实现

- [ ] todo

### mmap.c

用于内存映射，可将文件或设备映射到进程的地址空间中

未实现

### network.c

网络接口函数应该是

### poll.c

实现非阻塞的IO操作

未实现

### pow.c

实现了pow(x, y)函数，x的y次方

### printf.c

提供了许多printf函数

### pthread.c

提供了基于POSIX标准的pthread函数。

未实现

- [ ] todo

### pwd.c

获取用户信息

### resource.c

获取和设置进程的资源限制

其中获取进程的资源使用情况函数尚未实现

### sched.c

设置进程或线程的CPU亲和性函数

未实现

### select.c

用于IO多路复用，包括监视多个文件描述符上的事件。

### signal.c

Linux进程的信号机制，未实现

### socket.c

网络通信使用的socket函数。一部分未实现

### stat.c

文件系统中的一些操作函数，例如创建目录等。

未实现

### stdio.c

向屏幕输入输出的函数，其中输入未实现

### stdlib.c

随机数、绝对值等

快排等未实现。

- [ ] todo

### string.c

针对字符串的相关函数

### syslog.c

系统日志相关函数。

### time.c

关于时间的函数

未实现

### uio.c

不清楚，已实现

### unistd.c

不清楚，未实现

### utsname.c

未实现

### wait.c

未实现

## 问题

- [x] 禁用防火墙，可以访问github

