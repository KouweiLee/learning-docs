# ftrace-内核debug

ftrace可以追踪内核函数，可以追踪一个内核模块的所有函数调用。是一个很好用的内核debug神器。

## 问题汇总

在.config文件中加入`CONFIG_FTRACE=y`，可以让/sys/kernel/debug目录下出现tracing目录

参考链接：https://cloud.tencent.com/developer/article/2382403