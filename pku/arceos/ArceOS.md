# ArceOS

## 概述

ArceOS整体架构：

![image-20230725144720193](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202307251447277.png)

由以下3部分组成：

- apps: 应用程序。它的运行需要依赖于modules组件库。
- modules: ArceOS的组件库，与OS有关
- crates: 通用的基础库，为modules实现提供支持，与OS无关

### modules

ArceOS的组件库，与OS有关，modules列表：

- [axhal](http://rcore-os.cn/modules/axhal): ArceOS硬件抽象层，为特定平台的操作提供统一的API。它为指定的操作平台进行引导和初始化过程，并在硬件上提供有用的操作。
- [axruntime](http://rcore-os.cn/modules/axruntime): ArceOS 的运行时库，是应用程序运行的基础环境。

## 启动流程

### axhal

以下都是针对RISC-V架构的：

axhal/platform中特定的平台中，boot.rs的_start函数为ArceOS运行的第一段代码，bootloader在此前已经运行了。linker.lds.S规定了地址空间的布局，可以看到，text、data等段都位于内核空间。

`_start`：

调用`mod.rs/rust_entry()`: 做一些初始化工作，包括设置异常处理函数入口，并跳转到`axruntime/lib.rs/rust_main`

### axruntime

`lib.rs/rust_main`：接收两个参数，这两个参数都是`_start`接收的参数。最后会调用指定app中的main函数。

#### alloc

如果启用了alloc feature的话，会首先调用`init_allocator`：

memory_regions函数会返回内核区域和内核以外物理内存以里的区域，后者被标记为Free。

allocator就是针对被标记Free的内核以外物理内存以里的区域进行初始化的，这里的allocator就是指GLOBAL_ALLOCATOR。

首先初始化BitmapPageAllocator(palloc)，base为可分配内存区域的基地址，inner为一个位图，标记这些页均未分配。

之后初始化balloc。

alloc库的底层代码中，alloc、dealloc等，其实都调用了属性为#[*global_allocator*]的全局分配器，这个分配器需要实现GlobalAlloc特征，该特征中的alloc、dealloc方法就是alloc库真正调用的方法。在arceos中，该分配器为axalloc/lib.rs中的GLOBAL_ALLOCATOR。

## 设计实现hello_world程序

`_start()`通过链接会作为ArceOS运行的第一段代码

```
#[link_section = ".text.boot"]
unsafe extern "C" fn _start() -> ! { //ArceOS 运行的第一段代码
```

## 记录

每个app的main.rs文件中都包含如下命令：

```rust
#![no_std]
#![no_main]

#[no_mangle]
fn main()...
```

`no_std`表示rust会使用核心库，而不是标准库。核心库是不依赖于操作系统的库，而标准库则依赖于操作系统。

 `#![no_main]` 告诉编译器我们没有一般意义上的 `main` 函数，编译器不需要做初始化工作，否则会报错。

`#[no_mangle]`则告诉编译器不要修改该函数名，否则编译器一般会修改函数名，进而在与其他语言交互时会无法调用该函数。

## 小tip

1. 交叉编译

编译器运行的开发平台（x86_64）与可执行文件运行的目标平台（riscv-64）不同，我们把这称为 **交叉编译**



## 运行命令

清空：make clean A=apps/c/qsort_test

运行：make ARCH=riscv64 A=apps/c/qsort_test run

查看函数文档：man 函数类别 函数名。其中3表示库函数，2表示系统调用函数

合并分支main的代码到本分支：

git merge main
