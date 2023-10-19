# ArceOS文件系统

## Linux文件系统

Linux 文件系统会为每个文件分配两个数据结构：索引节点(index node)和目录项(directory entry)。

- 索引节点，也就是 *inode*，用来记录文件的元信息，比如 inode 编号、文件大小、访问权限、创建时间、修改时间、**数据在磁盘的位置**等等。索引节点是文件的**唯一**标识，它们之间一一对应，也同样都会被存储在硬盘中，所以**索引节点同样占用磁盘空间**。
- 目录项，也就是 *dentry*，用来记录文件的名字、**索引节点指针**以及与其他目录项的层级关联关系。多个目录项关联起来，就会形成目录结构，但它与索引节点不同的是，**目录项是由内核维护的一个数据结构，不存放于磁盘，而是缓存在内存**。

如果查询目录频繁从磁盘读，效率会很低，所以内核会把已经读过的目录用目录项这个数据结构缓存在内存，下次再次读到相同的目录时，只需从内存读就可以，大大提高了文件系统的效率。

![img](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308041602923.png)

磁盘会被分为3个存储区域：

- *超级块*，用来存储文件系统的详细信息，比如块个数、块大小、空闲块等等。
- *索引节点区*，用来存储索引节点；
- *数据块区*，用来存储文件或目录数据；

打开一个文件后，OS会跟踪进程打开的所有文件，每个进程会有一个打开文件表，表中每项代表文件描述符。

操作系统在打开文件表中维护着打开文件的状态和信息：

- 文件指针：系统跟踪上次读写位置作为当前文件位置指针，这种指针对打开文件的某个进程来说是唯一的；
- 文件打开计数器：文件关闭时，操作系统必须重用其打开文件表条目，否则表内空间不够用。因为多个进程可能打开同一个文件，所以系统在删除打开文件条目之前，必须等待最后一个进程关闭文件，该计数器跟踪打开和关闭的数量，当该计数为 0 时，系统关闭文件，删除该条目；
- 文件磁盘位置：绝大多数文件操作都要求系统修改文件数据，该信息保存在内存中，以免每个操作都从磁盘中读取；
- 访问权限：每个进程打开文件都需要有一个访问模式（创建、只读、读写、添加等），该信息保存在进程的打开文件表中，以便操作系统能允许或拒绝之后的 I/O 请求；

### 文件的存储

Inode中包含10个指向数据块的指针，和1个指向索引块的指针，索引块中包含指向n个数据块的指针。另外还包含指向二级、三级索引块的指针。

![早期 Unix 文件系统](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308041617718.png)

* 空闲空间的管理

采用位图法进行空闲空间管理。不仅用于数据空闲块，也用于inode空闲块。

* 目录的存储

目录也是文件，也有对应的inode。

目录文件的块中保存的是目录里一项一项的文件信息，通常采用列表的形式保存。如下图：

![目录格式哈希表](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308041630023.png)

通过inode号可以找到具体的文件了。为了便于查找文件，可以采用哈希表将文件名与inode进行对应。（这个图是不是错了？？？）

### 软链接与硬链接

通过硬链接和软链接可以给文件取别名：

* 硬链接（hard link）是**多个目录项中的「索引节点」指向一个文件**，也就是指向同一个 inode。由于多个目录项都是指向一个 inode，那么**只有删除文件的所有硬链接以及源文件时，系统才会彻底删除该文件。**

![硬链接](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308041635517.png)

* 软链接（symbolic link）则创建一个新的inode，inode所指向的内容实际上保存了一个绝对路径，如果这个绝对路径上的文件被删除了，那么就找不到了。

![软链接](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308041636920.png)

## axfs

是一个module。其中api提供了向上层应用的文件系统接口。

* 磁盘的建立

根据指定位置的磁盘镜像（将磁盘中所有内容复制到一个文件中），可创建出一个`RamDisk`. 

```rust
pub struct RamDisk {
    size: usize, // 磁盘大小
    data: Vec<u8>, // 磁盘内容
}
```

之后初始化scheduler（todo），然后初始化文件系统：

1. 根据RamDisk创建Disk

   ```rust
   /// A disk device with a cursor.
   pub struct Disk {
       block_id: u64,
       offset: usize,
       dev: AxBlockDevice,
   }
   ```

2. 根据Disk创建FatFileSystem：

   ```rust
   pub struct FatFileSystem {
       inner: fatfs::FileSystem<Disk, NullTimeProvider, LossyOemCpConverter>,// disk作为存储的fatfs
       root_dir: UnsafeCell<Option<VfsNodeRef>>, // None
   }
   ```

   FatFileSystem保存在main_fs中，之后初始化root_dir为fatfs的root_dir。

   main_fs（Arc<FatFileSystem\>）之后保存在根目录结构体中：

   ```rust
   struct RootDirectory {
       main_fs: Arc<dyn VfsOps>,
       mounts: Vec<MountPoint>,// 挂载的其他文件系统
   }
   ```

3. 挂载RamFS：

   ```rust
   pub struct RamFileSystem {
       parent: Once<VfsNodeRef>,
       root: Arc<DirNode>,
   }
   // RAM文件系统的目录节点
   pub struct DirNode {
       this: Weak<DirNode>,
       parent: RwLock<Weak<dyn VfsNodeOps>>,
       children: RwLock<BTreeMap<String, VfsNodeRef>>,
   }
   pub type VfsNodeRef = Arc<dyn VfsNodeOps>;
   ```

   

## axfs_vfs

[axfs_vfs - Rust (rcore-os.cn)](http://rcore-os.cn/arceos/axfs_vfs/index.html)

一个crate，表示虚拟文件系统接口。文件系统是文件和目录的集合，统称为nodes，类似于Linux中的inode。文件系统需要实现`VfsOps`特征，文件和目录需要实现`VfsNodeOps`特征。



## axfs_ramfs

