# QEMU模拟ARM64内核

## 一、安装交叉编译器 aarch64-none-linux-gnu- 10.3

网址：[https://developer.arm.com/downloads/-/gnu-a](https://developer.arm.com/downloads/-/gnu-a) 

工具选择：AArch64 GNU/Linux target (aarch64-none-linux-gnu) 

下载链接：[https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC) 

```bash
wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
ls gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

安装完成，记住路径，本机在：/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-

## 二、编译安装QEMU 7.2

1. wget [https://download.qemu.org/qemu-7.2.1.tar.xz](https://download.qemu.org/qemu-7.2.1.tar.xz)
2. tar 解压
3. mkdir build %% cd build
4. ../qemu-7.2.0/configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu
5. make -j2

## 三、编译Kernel 5.4

```bash
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4

make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

编译完毕，内核文件：arch/arm64/boot/Image

## 八、基于ubuntu 20.04 arm64 base构建文件系统

由于busybox制作的文件系统过于简单（如没有apt工具），因此，我们需要使用更丰富的ubuntu文件系统来定制。

下载：[ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)  

链接：[http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz

mkdir rootfs
# 创建一个ubuntu.img
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=8192 oflag=direct
mkfs.ext4 ubuntu-20.04-rootfs_ext4.img
# 将ubuntu.tar.gz放入已经挂载到rootfs上的ubuntu.img中
sudo mount -t ext4 ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/

# 让rootfs绑定和获取物理机的一些信息和硬件
sudo cp /usr/bin/qemu-system-aarch64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# 安装内核模块（可选）
cd linux-5.4
make ARCH=arm64 modules -j$(nproc) CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-
sudo make ARCH=arm64 modules_install CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- INSTALL_MOD_PATH=/root/rootfs

sudo chroot rootfs

apt-get update
apt-get install git sudo vim bash-completion -y
apt-get install net-tools ethtool ifupdown network-manager iputils-ping -y
apt-get install rsyslog resolvconf udev -y

# 如果上面软件包没有安装，至少要安装下面的包
apt-get install systemd -y

apt-get install build-essential git wget flex bison libssl-dev bc libncurses-dev kmod -y

adduser arm64
adduser arm64 sudo
echo "kernel-5_4" >/etc/hostname
echo "127.0.0.1 localhost" >/etc/hosts
echo "127.0.0.1 kernel-5_4">>/etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

然后模拟运行来验证：

```bash
qemu-system-aarch64 \
	-machine virt,gic_version=3 \
	-machine virtualization=true \
	-cpu cortex-a57 \
	-machine type=virt \
	-nographic \
	-smp 16 \
	-m 1024 \
	-kernel ./linux/arch/arm64/boot/Image \
	-append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
	-drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
	-device virtio-blk-device,drive=hd0 \
	-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
	-device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

**Q1**：执行`sudo chroot .`时，报错`chroot: failed to run command ‘/bin/bash’: Exec format error`

这是由于宿主机（一般是x86）与ubuntu-base（aarch64）的体系结构不兼容导致的。解决方法是安装qemu-user-static。

当主系统和chroot环境的体系结构不匹配时，qemu-user-static可以将chroot环境中的二进制文件转发给相应的QEMU仿真器进行处理和执行。使用qemu-user-static，可以在chroot环境中执行与chroot环境不匹配的二进制文件，而无需改变主系统的体系结构。

```bash
sudo apt-get install qemu-user-static
sudo update-binfmts --enable qemu-aarch64
```

**Q2**：在使用ubuntu-base启动虚拟aarch64平台时，等待dev-ttyAMA0.device超时而卡住，从而导致无法登录进入bash

```bash
sudo mount ubuntu-20.04-rootfs_ext4.img rootfs/
sudo chroot rootfs/
cp lib/systemd/system/serial-getty\@.service lib/systemd/system/serial-getty\@ttyAMA0.service
systemctl enable serial-getty\@ttyAMA0.service
exit
```

**修改内容**

```bash
sudo vim rootfs/lib/systemd/system/serial-getty@ttyAMA0.service
```

![Untitled](img/20230421edit.png)

将BindsTo和After开头的行注释掉

参考地址：

[systemd for Administrators, Part XVI](http://0pointer.de/blog/projects/serial-console.html)

最后卸载挂载，完成根文件系统的制作。

## 十、编译jailhouse

```bash
sudo mount ubuntu-20.04-rootfs_ext4.img rootfs/
```

编译rootfs：

```bash
# for x86
git clone https://github.com/siemens/jailhouse.git
cd jailhouse 
make

# for aarch64,进入rootfsjailhouse的文件夹
git clone https://github.com/siemens/jailhouse.git
cd jailhouse 
make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/korwylee/lgw/hypervisor/linux

sudo make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/korwylee/lgw/hypervisor/linux DESTDIR=/home/korwylee/lgw/hypervisor/linux install 
```

说明：编译jailhouse一定要指定KDIR，说明sysroot目标，才可以编译成功，并且需要提前编译一个linux作为sysroot，否则默认从本机linux源码目录中去找相应的库代码。

## 十一、运行jailhouse

```
cd ~/jailhouse
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
sudo ./tools/jailhouse cell create configs/arm64/qemu-arm64-gic-demo.cell
sudo ./tools/jailhouse cell load gic-demo inmates/demos/arm64/gic-demo.bin
sudo ./tools/jailhouse cell start gic-demo
sudo ./tools/jailhouse cell destroy gic-demo
sudo ./tools/jailhouse disable
```

* 启动non root linux

```
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/ram0 rdinit=/helloworld rw" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb -i ../initrd
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/ram0 rdinit=/linuxrc" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb -i ../init

ivsh memory
```

如果执行jailhouse报错说glibc版本低，则在rootfs中换源，可以增加22.04的清华源。步骤如下：

执行命令

```shell
sudo vi /etc/apt/sources.list
```

替换文件内容为：

```

```

替换后执行：

```
sudo apt update 
sudo apt install libc6
```



## 附：img扩容

当rootfs chroot空间不足时，需要扩容，按照以下步骤进行无损扩容：

```bash
# 首先取消挂载img
umount ./rootfs

## bug: umount: /root/rootfs: target is busy.
## 解决：umount ./rootfs -l 强行卸载（慎用）

dd if=/dev/zero of=add.img bs=1M count=4096 # 新建4G空间
cat add.img >> ubuntu-20.04-rootfs_ext4.img
e2fsck -f ubuntu-20.04-rootfs_ext4.img
resize2fs ubuntu-20.04-rootfs_ext4.img

mount ubuntu-20.04-rootfs_ext4.img rootfs
```

