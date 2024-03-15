# qemu-virtio-net连通外网配置

宿主机:

```
sudo ifconfig eth0 down
sudo brctl addbr br0 
sudo brctl addif br0 eth0
sudo brctl setfd br0 1
sudo ifconfig br0 0.0.0.0 promisc up 
sudo ifconfig eth0 0.0.0.0 promisc up 
sudo dhclient br0 

sudo tunctl -t tap0 -u root
sudo brctl addif br0 tap0  
sudo ifconfig tap0 0.0.0.0 promisc up
```



```
sudo ifconfig enp52s0 down    # 首先关闭宿主机网卡接口
sudo brctl addbr br0                     # 添加名为 br0 的网桥
sudo brctl addif br0 enp52s0        # 在 br0 中添加一个接口
sudo brctl stp br0 off                   # 如果只有一个网桥，则关闭生成树协议
sudo brctl setfd br0 1                   # 设置 br0 的转发延迟
sudo brctl sethello br0 1                # 设置 br0 的 hello 时间
sudo ifconfig br0 0.0.0.0 promisc up     # 启用 br0 接口
sudo ifconfig enp52s0 0.0.0.0 promisc up    # 启用网卡接口
sudo dhclient br0                        # 从 dhcp 服务器获得 br0 的 IP 地址
brctl show br0                      # 查看虚拟网桥列表
brctl showstp br0                   # 查看 br0 的各接口信息

sudo tunctl -t tap0 -u root
sudo brctl addif br0 tap0  
sudo ifconfig tap0 0.0.0.0 promisc up
```

网桥:

```
sudo brctl addbr br0
sudo brctl stp br0 on
sudo ip tuntap add name tap0 mode tap
sudo ip link set dev tap0 up
sudo brctl addif br0 tap0  
sudo dhclient br0
```



宿主机删除设备:

```
sudo brctl delif br0 tap0 
sudo tunctl -d tap0 
sudo brctl delif br0 enp52s0 
sudo ifconfig br0 down 
sudo brctl delbr br0 
sudo ifconfig enp52s0 up 
sudo dhclient -v enp52s0 # 使用DHCP 动态获取IP地址， -v  显示详情

```

## qemu成功连通外网配置

参考教程: [ubuntu下qemu虚拟机实现和主机以及互联网通信，图文详细教程](https://blog.csdn.net/qq_34160841/article/details/104901127)

首先执行modinfo tun, 只要显示有tun模块就行.

安装网桥管理包:

```
sudo apt install -y bridge-utils        # 虚拟网桥工具
sudo apt install -y uml-utilities       # UML（User-mode linux）工具
```

执行下述命令

```
sudo brctl addbr virbr0
sudo echo 'allow virbr0' >> /etc/qemu/bridge.conf
sudo brctl stp virbr0 on
sudo ip tuntap add name virbr0-nic mode tap
sudo ip link set dev virbr0-nic up
sudo brctl addif virbr0 virbr0-nic
sudo systemctl enable libvirtd  #开启启动网桥
sudo dhclient virbr0
```

启动qemu参数增加:

```
    -net nic,macaddr=aa:e6:1d:01:01:01,model=e1000e  \
    -net bridge,id=net0,helper=/usr/lib/qemu/qemu-bridge-helper,br=virbr0
```

在虚拟机中, ping百度即可. 如果ping不通, 可以重启电脑再试. 另外不知道无线网卡是否行, 我用的是有线网卡. 

## root linux的配置

在编译root linux的镜像前, 在.config文件中把IPv6和BRIDGE的config都改成y

以下的废弃掉了

- [ ] ~~接下来设计网络图, 写一个脚本文件~~

```
arm64@kernel-54:~/hvisor$ sudo brctl addbr br0
[sudo] password for arm64:
arm64@kernel-54:~/hvisor$ sudo brctl addif br0 enp0s1
arm64@kernel-54:~/hvisor$ ifconfig enp0s1 0
arm64@kernel-54:~/hvisor$ dhclient br0
arm64@kernel-54:~/hvisor$ sudo ip tuntap add dev tap0 mode tap
arm64@kernel-54:~/hvisor$ sudo brctl addif br0 tap0
arm64@kernel-54:~/hvisor$ sudo ip link set dev tap0 up
```

***

可以通过systemd-networkd来自动根据配置文件来创建网络拓扑. 在/etc/systemd/network/目录下, 新建4个文件:

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
sudo dhclient eth0
```

