# mmap相关问题

[【分享】Linux保留内存的对齐访问问题。 (xilinx.com)](https://support.xilinx.com/s/question/0D52E00006hpqKISAY/分享linux保留内存的对齐访问问题?language=zh_CN)

[mmap 内存与 memcpy 总线错误：解决难题的妙招 - ByteZoneX社区](https://www.bytezonex.com/archives/RexA4PGg.html)

## /dev/mem的问题

通过cat /proc/[pid]/smaps和/proc/[pid]/maps查看内存。

virtio bridge：

```
ffffb161f000-ffffb1620000 rw-s 00000000 00:06 242                        /dev/hvisor
Size:                  4 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                   0 kB
Pss:                   0 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:            0 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:        0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:		0
VmFlags: rd wr sh mr mw me ms pf io de dd 
```

non root mem：

```
ffff81430000-ffffb1430000 rw-s 50000000 00:06 30                         /dev/mem
Size:             786432 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                   0 kB
Pss:                   0 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:            0 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:        0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:		0
VmFlags: rd wr sh mr mw me ms pf io de dd 
```



## TODO

- [ ] no-map这个参数到底会怎么样，在设备树里加不加
- [ ] 如何能使mmap随意在内存里读写
- [ ] operation not permitted when mmap is why？

现在确定是因为不加no-map这个导致的。。。