# mmap相关问题

[【分享】Linux保留内存的对齐访问问题。 (xilinx.com)](https://support.xilinx.com/s/question/0D52E00006hpqKISAY/分享linux保留内存的对齐访问问题?language=zh_CN)

[mmap 内存与 memcpy 总线错误：解决难题的妙招 - ByteZoneX社区](https://www.bytezonex.com/archives/RexA4PGg.html)

## TODO

- [ ] no-map这个参数到底会怎么样，在设备树里加不加
- [ ] 如何能使mmap随意在内存里读写
- [ ] operation not permitted when mmap is why？

现在确定是因为不加no-map这个导致的。。。