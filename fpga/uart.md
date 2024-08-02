# uart

不对啊，这个是说uart用IP核的方式实现吗？？？

uart2在PL Bank49中。

53页

![image-20240521085803955](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405210858052.png)

CP2108是一个usb转接uart的桥接芯片，能支持4个channel同时工作，连在电脑上时会显示为4个串行端口。其中channel 2为PL端的uart。