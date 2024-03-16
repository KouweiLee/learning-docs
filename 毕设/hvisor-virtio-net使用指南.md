# hvisor-virtio-net使用指南

## qemu成功连通外网配置

首先执行modinfo tun, 只要显示有tun模块就行. 否则需要安装tun模块. 

启动qemu参数增加:

```
-netdev user,id=n0,hostfwd=tcp::5555-:22 -device virtio-net-device,bus=virtio-mmio-bus.24,netdev=n0
```

在虚拟机中, ping百度即可. 如果ping不通, 可以重启电脑再试. 

## root linux的配置

在编译root linux的镜像前, 在.config文件中把IPv6和BRIDGE的config都改成y, 以支持在root linux中创建网桥和tap设备. 

启动root linux后, 通过systemd-networkd来自动根据配置文件来创建网络拓扑. 在/etc/systemd/network/目录下, 新建4个文件:

50-hvisor.netdev

```
[NetDev]
Name=br0
Kind=bridge
```

50-hvisor.network

```
[Match]
Name=e* tap*

[Network]
Bridge=acrn-br0
```

50-tap0.netdev

```
[NetDev]
Name=tap0
Kind=tap
```

50-eth.network

```
[Match]
Name=br0

[Network]
DHCP=ipv4
```

这四个文件描述了类似于下图的网络拓扑:

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240315112839357.png" alt="image-20240315112839357" style="zoom:50%;" />

其中eth0和tap设备, 成为了br0的一个端口(或者叫网线, eth0连接着外部网络, tap0连接着virtio-net device0虚拟出来的网卡), 因此这两个设备都不需要ip地址了, br0和virtio-net device0有ip地址即可. 

之后执行:

```bash
sudo systemctl enable --now systemd-networkd
```

这会自动在开机时就根据这配置文件来创建网络拓扑.

## non root linux的配置

non root linux需要手动激活虚拟网卡. 方法是:

```c
// 查看可用的网络设备
ip link
// 激活eth0网卡
sudo ip link set eth0 up
// 为eth0分配ip地址
sudo dhclient eth0
```

## 参考资料

1. [ubuntu下qemu虚拟机实现和主机以及互联网通信，图文详细教程](https://blog.csdn.net/qq_34160841/article/details/104901127)

