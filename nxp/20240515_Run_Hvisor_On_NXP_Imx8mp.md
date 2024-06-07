# 在NXP IMX8MP开发板上运行hvisor

时间：2024/5/15

作者：陈林锟

摘要：介绍如何在在NXP IMX8MP开发板上运行hvisor

## 下载开发板资料

https://pan.baidu.com/s/1XimrhPBQIG5edY4tPN9_pw?pwd=kdtk
提取码：kdtk

## 解压Linux SDK

首先进入开发板资料目录OKMX8MP-C_Linux5.4.70_Qt5.15.0。

```bash
cd Linux/sources

# 合并分卷压缩包
cat OK8MP-linux-sdk.tar.bz2.0* > OK8MP-linux-sdk.tar.bz2

# 解压合并的压缩包
tar -xvjf OK8MP-linux-sdk.tar.bz2
```

执行完这些步骤后，Linux SDK 将会解压到当前目录。

## 搭建 TFTP 服务器

方便开发板与主机间的数据传输，具体步骤如下：

1. 安装 TFTP 服务器软件包

    ```bash
    sudo apt-get update
    sudo apt-get install tftpd-hpa tftp-hpa
    ```

2. 配置 TFTP 服务器

    创建 TFTP 根目录并设置权限：

    ```bash
    mkdir -p ~/tftp
    sudo chown -R $USER:$USER ~/tftp
    sudo chmod -R 755 ~/tftp
    ```
    编辑 tftpd-hpa 配置文件：

    ```bash
    sudo nano /etc/default/tftpd-hpa
    ```
    修改如下：

    ```plaintext
    # /etc/default/tftpd-hpa

    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/home/<your-username>/tftp"
    TFTP_ADDRESS=":69"
    TFTP_OPTIONS="-l -c -s"
    ```
    将 `<your-username>` 替换为实际用户名。

3. 启动/重启 TFTP 服务

    ```bash
    sudo systemctl restart tftpd-hpa
    ```

4. 验证 TFTP 服务器

    ```bash
    echo "TFTP Server Test" > ~/tftp/testfile.txt
    ```

    ```bash
    tftp localhost
    tftp> get testfile.txt
    tftp> quit
    cat testfile.txt
    ```
    若显示 "TFTP Server Test"，则 TFTP 服务器工作正常。

## 安装交叉编译工具链

1. 下载交叉编译工具链：
    ```bash
    wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
    ```

2. 解压工具链：
    ```bash
    tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
    ```

3. 添加路径，使 `aarch64-none-linux-gnu-*` 可以直接使用，修改 `~/.bashrc` 文件：
    ```bash
    echo 'export PATH=$PWD/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH' >> ~/.bashrc
    source ~/.bashrc
    ```

## 编译 Linux 内核（可选）

> 这一步是可选的，也就是说也可以不编译内核，直接使用`Linux/Images/kernel/Image`。但是如果后续需要调试或者修改内核，则需要做这一步！

以下步骤用于下载并安装适用于 ARM 的交叉编译工具链，初始化 Git 仓库，并编译 Linux 内核。编译完成后，内核镜像会被复制到 `~/tftp` 目录，方便后续使用。

1. 切换到 Linux 内核源码目录：
    ```bash
    cd Linux/sources/OK8MP-linux-sdk
    ```

2. 初始化 Git 仓库并提交初始代码：
    ```bash
    git init
    git add .
    git commit -m "Initial commit"
    ```

3. 新建一个 `compile.sh` 脚本，内容如下：
    ```bash
    #!/bin/bash
    
    # 设置 Linux 内核配置
    make OK8MP-C_defconfig ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu-
    
    # 编译 Linux 内核
    make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- Image -j$(nproc)
    
    # 创建 tftp 目录（如果不存在）
    mkdir -p ~/tftp
    
    # 复制编译后的镜像到 tftp 目录
    cp arch/arm64/boot/Image ~/tftp/
    ```

4. 确保 `compile.sh` 可执行：
    ```bash
    chmod +x compile.sh
    ```

5. 运行 `compile.sh` 编译 Linux 内核，并将编译后的镜像复制到 tftp 目录：
    ```bash
    ./compile.sh
    ```

6. 配置开机自启tftpd-hpa：
	```bash
	sudo systemctl enable tftpd-hpa
	```

## 编译hvisor

1. 切换到 hvisor 源码目录：
    ```bash
    cd <hvisor-source-dir>
    ```

2. 切换到 `dev-nxp` 分支：
	```bash
	git checkout dev-nxp
	```

3. 编译hvisor：
	```
	make LOG=<trace/debug/info/warn/error> cp 
	```

## 构建SD卡启动盘

1. 将SD卡插入读卡器，并连接至主机。
2. 切换至Linux/Images目录。
3. 执行以下命令，进行分区：

    ```bash
    fdisk <$DRIVE>
    d  # 删除所有分区
    n  # 创建新分区
    p  # 选择主分区
    1  # 分区编号为1
    16384  # 起始扇区
    t  # 更改分区类型
    83  # 选择Linux文件系统（ext4）
    w  # 保存并退出
    ```

4. 将启动文件写入SD卡启动盘：
    ```bash
    dd if=imx-boot_1G.bin of=<$DRIVE> bs=1K seek=32 conv=fsync
    ```

5. 格式化SD卡启动盘的第一个分区为ext4格式：
    ```bash
    mkfs.ext4 <$DRIVE>1
    ```

6. 将SD卡读卡器拔出，重新连接。将Buildroot的rootfs.ext4解压到SD卡1号分区。

	```bash
	tar -xvf rootfs.tar -C <path/to/mounted/SD/card/partition>
	```

rootfs.tar下载地址：
```
https://disk.pku.edu.cn/link/AADFFFE8F568DE4E73BE24F5AED54B00EB
文件名：rootfs.tar
```

7. 完成后，弹出SD卡。

## 配置有线网连接

1. 使用网线将开发板的网口（共有两个，请选择下方的一个）与主机连接。
2. 配置主机有线网卡，ip：192.169.137.2, netmask: 255.255.255.0。

## 拷贝必要文件到tftp根目录

1. 拷贝hvisor镜像（`hvisor.bin`）到tftp根目录：在编译hvisor时已经完成（`make LOG=<log_level> cp`）
2. 拷贝Guest Linux Image：在编译linux时已经完成；如果没有手动编译内核，也可以拷贝`Linux/Images/kernel/Image`。
3. 拷贝设备树文件：OK8MP-C.dtb（位置在`Linux/Images/kernel/OK8MP-C.dtb`）

如果正确完成以上操作，tftp根目录下的内容应至少包括：

```
hvisor.bin   # hvisor镜像
OK8MP-C.dtb  # 给hvisor提供的设备树文件
Image        # Guest Linux镜像
linux1.dtb   # Root Linux的设备树文件
```

## 连接开发板（设置拨码开关、SD卡、串口和有线网）

1. 调整拨码开关以启用SD卡启动模式：(1,2,3,4) = (ON,ON,OFF,OFF)。
2. 将SD卡插入SD插槽。
3. 使用串口线将开发板与主机相连。
4. 将有线网线连接至开发板的网口。

## 启动hvisor

1. 安装gtkterm: `sudo apt-get install gtkterm`
2. gtkterm连接串口/dev/ttyAMA0（或者是/dev/ttyAMA1）
3. 重启开发板，如果连接和配置正确，gtkterm应有开发板的串口输出。
4. 刚开始进入uboot，立刻多按几次空格，进入uboot命令行模式。
5. 在shell输入以下命令，按回车键执行：

```bash
setenv serverip 192.169.137.2; setenv ipaddr 192.169.137.3; setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0x50000000; setenv zone0_fdt_addr 0x70000000; tftp ${loadaddr} ${serverip}:hvisor.bin; tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb; tftp ${zone0_kernel_addr} ${serverip}:Image; tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

```
setenv serverip 172.28.3.15; dhcp; setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0x50000000; setenv zone0_fdt_addr 0x70000000; tftp ${loadaddr} ${serverip}:hvisor.bin; tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb; tftp ${zone0_kernel_addr} ${serverip}:Image; tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

解释:

- `setenv serverip 192.169.137.2`：设置tftp服务器的IP地址。
- `setenv ipaddr 192.169.137.3`：设置开发板的IP地址。
- `setenv loadaddr 0x40400000`：设置hvisor镜像的加载地址。
- `setenv fdt_addr 0x40000000`：设置设备树文件的加载地址。
- `setenv zone0_kernel_addr 0x50000000`：设置guest Linux镜像的加载地址。
- `setenv zone0_fdt_addr 0x70000000`：设置root Linux的设备树文件的加载地址。
- `tftp ${loadaddr} ${serverip}:hvisor.bin`：从tftp服务器下载hvisor镜像到hvisor的加载地址。
- `tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb`：从tftp服务器下载设备树文件到设备树文件的加载地址。
- `tftp ${zone0_kernel_addr} ${serverip}:Image`：从tftp服务器下载guest Linux镜像到guest Linux镜像的加载地址。
- `tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb`：从tftp服务器下载root Linux的设备树文件到root Linux的设备树文件的加载地址。
- `bootm ${loadaddr} - ${fdt_addr}`：启动hvisor，加载hvisor镜像和设备树文件。

6. 等待hvisor启动，如果没有报错，则hvisor已经启动成功。

## JTAG调试

1. 从官网(https://www.segger.com/downloads/jlink/JLink_Linux_V796h_x86_64.deb)下载JLink软件包
2. 安装jlink：
    ```bash 
    sudo dpkg -i JLink_Linux_V796h_x86_64.deb
    ```
3. 连接开发板的JTAG接口。
4. 在hvisor目录下创建2个terminal，分别运行`make jlink-server`和`make monitor`，即可启动调试。

注意：连接/断开JTAG一定要在断电的情况下进行，以防烧坏板子！！！

