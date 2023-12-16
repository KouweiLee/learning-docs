# 手动实现一个Vec

## Layout

要实现一个Vec，首先要明确它的整体布局：一个指向堆上数据的指针、容量、现有元素个数：

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct Vec<T> {
    ptr: NonNull<T>,
    cap: usize,
    len: usize,
}

unsafe impl<T: Send> Send for Vec<T> {}
unsafe impl<T: Sync> Sync for Vec<T> {}
```

其中ptr可以采用*mut T。但是可以通过NonNull来加强一下约束，使得ptr必须不能为空指针。另外如果T是Send和Sync时，为Vec也标记相应的属性。



## 参考资料

[Layout - The Rustonomicon (rust-lang.org)](https://doc.rust-lang.org/nomicon/vec/vec-layout.html)：教程