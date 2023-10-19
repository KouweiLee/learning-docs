# IO MMU

[Input–output memory management unit - Wikipedia](https://en.m.wikipedia.org/wiki/Input–output_memory_management_unit)

IOMMU是一个连接DMA总线和主存的内存管理单元。它将外设虚拟地址映射为主存的物理地址。

![image-20230714110327654](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202307141103706.png)

## 优点

相比DMA的优点为：

1. 主存中可以分配大片不连续的内存，IOMMU可以将外设连续的虚拟地址映射到离散的物理地址中

2. 当外设由于地址空间不够长，不支持直接对整个物理地址空间取址时，通过IOMMU仍然可以对整个物理内存取址

3. 可以防止恶意外设进行攻击，或者错误外设尝试错误地数据传输。OS是控制MMU和IOMMU的唯一控制者，IO MMU的映射表对外设不可见。

4. **虚拟化：**当客户机使用硬件外设时，硬件可能是通过DMA直接访问内存（硬件通过DMA发出物理地址），但由于硬件只知道guest物理地址，但不知道host物理地址，也不知道guest物理地址到host物理地址的映射关系，可能会造成错误。如果hypervisor干预IO操作来实现映射则可以避免这个问题，但会带来IO操作的延时。

   IOMMU可以很好地解决这个问题，通过将硬件请求的guest物理地址重映射到host物理地址。

## 缺点

相比DMA的缺点是：

1. IOMMU地址翻译和管理会带来性能开销
2. IO页表会带来内存开销

## 问题

外设为什么使用的是guest物理地址？