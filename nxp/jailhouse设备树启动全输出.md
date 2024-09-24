

```
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 5.4.70-2.3.0-g7c70c4f0d-dirty (lgw@LG) (gcc version 10.3.1 20210621 (GNU Toolchain for the A-profile Architecture 10.3-2021.07 (arm-10.29))) #72 SMP PREEMPT Mon Sep 16 15:26:18 CST 2024
[    0.000000] Machine model: Forlinx OK8MPlus-C board
[    0.000000] earlycon: ec_imx6q0 at MMIO 0x0000000030890000 (options '115200')
[    0.000000] printk: bootconsole [ec_imx6q0] enabled
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] OF: reserved mem: failed to allocate memory for node 'linux,cma'
[    0.000000] Reserved memory: created DMA memory pool at 0x0000000055000000, size 0 MiB
[    0.000000] OF: reserved mem: initialized node vdev0vring0@55000000, compatible id shared-dma-pool
[    0.000000] Reserved memory: created DMA memory pool at 0x0000000055008000, size 0 MiB
[    0.000000] OF: reserved mem: initialized node vdev0vring1@55008000, compatible id shared-dma-pool
[    0.000000] Reserved memory: created DMA memory pool at 0x0000000055400000, size 1 MiB
[    0.000000] OF: reserved mem: initialized node vdevbuffer@55400000, compatible id shared-dma-pool
[    0.000000] cma: Reserved 320 MiB at 0x00000000bc000000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000050000000-0x00000000cfffffff]
[    0.000000] NUMA: NODE_DATA [mem 0xbbc44500-0xbbc45fff]
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000050000000-0x00000000cfffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000050000000-0x0000000054ffffff]
[    0.000000]   node   0: [mem 0x0000000055010000-0x00000000550fefff]
[    0.000000]   node   0: [mem 0x0000000055100000-0x00000000553fffff]
[    0.000000]   node   0: [mem 0x0000000055500000-0x00000000557fffff]
[    0.000000]   node   0: [mem 0x0000000056000000-0x000000005fffffff]
[    0.000000]   node   0: [mem 0x0000000070000000-0x00000000923fffff]
[    0.000000]   node   0: [mem 0x0000000094400000-0x00000000cfffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000050000000-0x00000000cfffffff]
[    0.000000] psci: probing for conduit method from DT.
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x84000000, arg0:0x0, arg1:0x0, arg2:0x0
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x84000006, arg0:0x0, arg1:0x0, arg2:0x0
[    0.000000] psci: Trusted OS migration not required
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x8400000a, arg0:0x80000000, arg1:0x0, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x80000000, arg0:0x0, arg1:0x0, arg2:0x0
[    0.000000] psci: SMC Calling Convention v1.0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x8400000a, arg0:0xc4000001, arg1:0x0, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x8400000a, arg0:0xc400000e, arg1:0x0, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0x8400000a, arg0:0xc4000012, arg1:0x0, arg2:0x0
[    0.000000] percpu: Embedded 24 pages/cpu s58904 r8192 d31208 u98304
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: detected: ARM erratum 845719
[    0.000000] CPU features: detected: GIC system register CPU interface
[    0.000000] CPU features: kernel page table isolation forced ON by KASLR
[    0.000000] CPU features: detected: Kernel page table isolation (KPTI)
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 441235
[    0.000000] Policy zone: DMA32
[    0.000000] Kernel command line: clk_ignore_unused console=ttymxc1,115200 earlycon=ec_imx6q,0x30890000,115200 root=/dev/mmcblk2p2 rootwait rw
[    0.000000] Dentry cache hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[    0.000000] Inode-cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 1380108K/1792956K available (17020K kernel code, 1260K rwdata, 6576K rodata, 2880K init, 1010K bss, 85168K reserved, 327680K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
[    0.000000] 	Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: 160 SPIs implemented
[    0.000000] GICv3: 0 Extended SPIs implemented
[    0.000000] GICv3: Distributor has no Range Selector support
[    0.000000] GICv3: 16 PPIs implemented
[    0.000000] GICv3: no VLPI support, no direct LPI support
[    0.000000] GICv3: CPU0: found redistributor 0 region 0:0x0000000038880000
[    0.000000] ITS: No ITS available, not enabling LPIs
[    0.000000] random: get_random_bytes called from start_kernel+0x2b4/0x448 with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 8.00MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1d854df40, max_idle_ns: 440795202120 ns
[    0.000003] sched_clock: 56 bits at 8MHz, resolution 125ns, wraps every 2199023255500ns
[    0.008610] Console: colour dummy device 80x25
[    0.012564] Calibrating delay loop (skipped), value calculated using timer frequency.. 16.00 BogoMIPS (lpj=32000)
[    0.022844] pid_max: default: 32768 minimum: 301
[    0.027536] LSM: Security Framework initializing
[    0.032160] Mount-cache hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.039562] Mountpoint-cache hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.048412] ASID allocator initialised with 32768 entries
[    0.052928] rcu: Hierarchical SRCU implementation.
[    0.058891] EFI services will not be available.
[    0.062342] smp: Bringing up secondary CPUs ...
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc4000003, arg0:0x1, arg1:0xa14961e0, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:307) psci: try to wake up cpu 1
[INFO  1] (hvisor::arch::aarch64::cpu:155) cpu 1 started
[    0.101522] Detected VIPT I-cache on CPU1
[    0.101577] GICv3: CPU1: found redistributor 1 region 0:0x00000000388a0000
[    0.101699] CPU1: Booted secondary processor 0x0000000001 [0x410fd034]
[    0.101795] smp: Brought up 1 node, 2 CPUs
[    0.120550] SMP: Total of 2 processors activated.
[    0.125269] CPU features: detected: 32-bit EL0 Support
[    0.130442] CPU features: detected: CRC32 instructions
[    0.142335] CPU: All CPU(s) started at EL1
[    0.143587] alternatives: patching kernel code
[    0.148853] devtmpfs: initialized
[    0.159686] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.166633] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.182664] pinctrl core: initialized pinctrl subsystem
[    0.185759] DMI not present or invalid.
[    0.189112] NET: Registered protocol family 16
[    0.200444] DMA: preallocated 256 KiB pool for atomic allocations
[    0.203730] audit: initializing netlink subsys (disabled)
[    0.209350] audit: type=2000 audit(0.148:1): state=initialized audit_enabled=0 res=1
[    0.216952] cpuidle: using governor menu
[    0.221319] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.228575] Serial: AMBA PL011 UART driver
[    0.231849] imx mu driver is registered.
[    0.235737] imx rpmsg driver is registered.
[    0.244256] imx8mp-pinctrl 30330000.pinctrl: initialized IMX pinctrl driver
[    0.261780] irq: type mismatch, failed to map hwirq-64 for interrupt-controller@38800000!
[    0.279767] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.283667] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.290382] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.297116] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.304786] cryptd: max_cpu_qlen set to 1000
[    0.311565] ACPI: Interpreter disabled.
[    0.313610] iommu: Default domain type: Translated 
[    0.317594] vgaarb: loaded
[    0.320390] SCSI subsystem initialized
[    0.324224] usbcore: registered new interface driver usbfs
[    0.329485] usbcore: registered new interface driver hub
[    0.334815] usbcore: registered new device driver usb
[    0.341202] mc: Linux media interface: v0.10
[    0.344162] videodev: Linux video capture interface: v2.00
[    0.349705] pps_core: LinuxPPS API ver. 1 registered
[    0.354639] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.363834] PTP clock support registered
[    0.367934] EDAC MC: Ver: 3.0.0
[    0.371802] No BMan portals available!
[    0.374921] QMan: Allocated lookup table at (____ptrval____), entry count 65537
[    0.382373] No QMan portals available!
[    0.386307] No USDPAA memory, no 'fsl,usdpaa-mem' in device-tree
[    0.392377] FPGA manager framework
[    0.395294] Advanced Linux Sound Architecture Driver Initialized.
[    0.401695] Bluetooth: Core ver 2.22
[    0.404961] NET: Registered protocol family 31
[    0.409417] Bluetooth: HCI device and connection manager initialized
[    0.415805] Bluetooth: HCI socket layer initialized
[    0.420703] Bluetooth: L2CAP socket layer initialized
[    0.425783] Bluetooth: SCO socket layer initialized
[    0.431397] clocksource: Switched to clocksource arch_sys_counter
[    0.436941] VFS: Disk quotas dquot_6.6.0
[    0.440780] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.447780] pnp: PnP ACPI: disabled
[    0.457076] thermal_sys: Registered thermal governor 'step_wise'
[    0.457080] thermal_sys: Registered thermal governor 'power_allocator'
[    0.460419] OF: /thermal-zones/cpu-thermal/cooling-maps/map0: could not find phandle
[    0.474605] thermal_sys: failed to build thermal zone cpu-thermal: -22
[    0.481353] NET: Registered protocol family 2
[    0.485818] tcp_listen_portaddr_hash hash table entries: 1024 (order: 2, 16384 bytes, linear)
[    0.494159] TCP established hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.502183] TCP bind hash table entries: 16384 (order: 6, 262144 bytes, linear)
[    0.509632] TCP: Hash tables configured (established 16384 bind 16384)
[    0.516055] UDP hash table entries: 1024 (order: 3, 32768 bytes, linear)
[    0.522767] UDP-Lite hash table entries: 1024 (order: 3, 32768 bytes, linear)
[    0.530018] NET: Registered protocol family 1
[    0.534611] RPC: Registered named UNIX socket transport module.
[    0.540235] RPC: Registered udp transport module.
[    0.544953] RPC: Registered tcp transport module.
[    0.549676] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.556489] PCI: CLS 0 bytes, default 64
[    0.560681] hw perfevents: enabled with armv8_pmuv3 PMU driver, 7 counters available
[    0.568258] kvm [1]: HYP mode not available
[    0.574946] Initialise system trusted keyrings
[    0.576662] workingset: timestamp_bits=44 max_order=19 bucket_order=0
[    0.588141] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.591714] NFS: Registering the id_resolver key type
[    0.596234] Key type id_resolver registered
[    0.600414] Key type id_legacy registered
[    0.604444] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.611188] jffs2: version 2.2. (NAND) © 2001-2006 Red Hat, Inc.
[    0.617630] 9p: Installing v9fs 9p2000 file system support
[    0.635589] Key type asymmetric registered
[    0.636831] Asymmetric key parser 'x509' registered
[    0.641760] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 244)
[    0.649175] io scheduler mq-deadline registered
[    0.653723] io scheduler kyber registered
[    0.659839] samsung-hdmi-phy 32fdff00.hdmiphy: failed to get phy apb clk: -517
[    0.669196] pwm-backlight lvds_backlight: lvds_backlight supply power not found, using dummy regulator
[    0.675834] pwm-backlight dsi_backlight: dsi_backlight supply power not found, using dummy regulator
[    0.685412] EINJ: ACPI disabled.
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000005, arg0:0x1, arg1:0x0, arg2:0x0
[    0.708306] i.MX8MP clock driver probe done
[    0.713774] imx-sdma 30bd0000.dma-controller: Direct firmware load for imx/sdma/sdma-imx7d.bin failed with error -2
[    0.717026] mxs-dma 33000000.dma-apbh: initialized
[    0.721437] imx-sdma 30bd0000.dma-controller: Falling back to sysfs fallback for: imx/sdma/sdma-imx7d.bin
[    0.727501] Bus freq driver module loaded
[    0.745661] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    0.751346] 30860000.serial: ttymxc0 at MMIO 0x30860000 (irq = 27, base_baud = 5000000) is a IMX
[    0.758470] 30880000.serial: ttymxc2 at MMIO 0x30880000 (irq = 28, base_baud = 5000000) is a IMX
[    0.767275] 30890000.serial: ttymxc1 at MMIO 0x30890000 (irq = 29, base_baud = 1500000) is a IMX
[    0.775669] printk: console [ttymxc1] enabled
[    0.775669] printk: console [ttymxc1] enabled
[    0.784304] printk: bootconsole [ec_imx6q0] disabled
[    0.784304] printk: bootconsole [ec_imx6q0] disabled
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x11, arg2:0x1
[    0.810071] imx-lcdifv3 32fc6000.lcd-controller: No irq get
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x11, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x11, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x12, arg2:0x1
[    0.856941] imx-hdmi-pavi 32fc4000.hdmi-pai-pvi: No pvi clock get
[    0.870086] brd: module loaded
[    0.877781] loop: module loaded
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[    0.935684] imx_hdmimix_clk_probe
[    0.939008] imx_hdmimix_clk_probe: get dummy
[    0.947847] imx ahci driver is registered.
[    0.960916] random: fast init done
[    0.966016] spi-nor spi0.0: w25q128 (16384 Kbytes)
[    0.977278] libphy: Fixed MDIO Bus: probed
[    0.982031] tun: Universal TUN/TAP device driver, 1.6
[    0.987959] thunder_xcv, ver 1.0
[    0.991218] thunder_bgx, ver 1.0
[    0.994478] nicpf, ver 1.0
[    0.998024] pps pps0: new PPS source ptp0
[    1.007982] Freescale FM module, FMD API version 21.1.0
[    1.013526] Freescale FM Ports module
[    1.017194] fsl_mac: fsl_mac: FSL FMan MAC API based driver
[    1.022957] fsl_dpa: FSL DPAA Ethernet driver
[    1.027464] fsl_advanced: FSL DPAA Advanced drivers:
[    1.032436] fsl_proxy: FSL DPAA Proxy initialization driver
[    1.038172] fsl_oh: FSL FMan Offline Parsing port driver
[    1.044506] hclge is initializing
[    1.047833] hns3: Hisilicon Ethernet Network Driver for Hip08 Family - version
[    1.055060] hns3: Copyright (c) 2017 Huawei Corporation.
[    1.060427] e1000: Intel(R) PRO/1000 Network Driver - version 7.3.21-k8-NAPI
[    1.067480] e1000: Copyright (c) 1999-2006 Intel Corporation.
[    1.073266] e1000e: Intel(R) PRO/1000 Network Driver - 3.2.6-k
[    1.079104] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    1.085054] igb: Intel(R) Gigabit Ethernet Network Driver - version 5.6.0-k
[    1.092019] igb: Copyright (c) 2007-2014 Intel Corporation.
[    1.097620] igbvf: Intel(R) Gigabit Virtual Function Network Driver - version 2.4.0-k
[    1.105454] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    1.111575] sky2: driver version 1.30
[    1.115868] imx-dwmac 30bf0000.ethernet: IRQ eth_lpi not found
[    1.122223] PPP generic driver version 2.4.2
[    1.126633] PPP BSD Compression module registered
[    1.131347] PPP Deflate Compression module registered
[    1.136416] PPP MPPE Compression module registered
[    1.141212] NET: Registered protocol family 24
[    1.145695] GobiNet: Quectel_Linux&Android_GobiNet_Driver_V1.6.2.14
[    1.152003] usbcore: registered new interface driver GobiNet
[    1.157693] usbcore: registered new interface driver asix
[    1.163134] usbcore: registered new interface driver ax88179_178a
[    1.169255] usbcore: registered new interface driver cdc_ether
[    1.175115] usbcore: registered new interface driver net1080
[    1.180809] usbcore: registered new interface driver zaurus
[    1.186419] usbcore: registered new interface driver cdc_ncm
[    1.192106] usbcore: registered new interface driver huawei_cdc_ncm
[    1.198574] VFIO - User Level meta-driver version: 0.3
[    1.208255] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    1.214801] ehci-pci: EHCI PCI platform driver
[    1.219294] ehci-platform: EHCI generic platform driver
[    1.224740] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    1.230945] ohci-pci: OHCI PCI platform driver
[    1.235426] ohci-platform: OHCI generic platform driver
[    1.241313] usbcore: registered new interface driver cdc_wdm
[    1.247136] usbcore: registered new interface driver uas
[    1.252492] usbcore: registered new interface driver usb-storage
[    1.258557] usbcore: registered new interface driver usbserial_generic
[    1.265103] usbserial: USB Serial support registered for generic
[    1.271147] usbcore: registered new interface driver ftdi_sio
[    1.276911] usbserial: USB Serial support registered for FTDI USB Serial Device
[    1.284249] usbcore: registered new interface driver option
[    1.289839] usbserial: USB Serial support registered for GSM modem (1-port)
[    1.296827] usbcore: registered new interface driver usb_serial_simple
[    1.303372] usbserial: USB Serial support registered for carelink
[    1.309481] usbserial: USB Serial support registered for zio
[    1.315159] usbserial: USB Serial support registered for funsoft
[    1.321186] usbserial: USB Serial support registered for flashloader
[    1.327559] usbserial: USB Serial support registered for google
[    1.333495] usbserial: USB Serial support registered for libtransistor
[    1.340037] usbserial: USB Serial support registered for vivopay
[    1.346059] usbserial: USB Serial support registered for moto_modem
[    1.352342] usbserial: USB Serial support registered for motorola_tetra
[    1.358976] usbserial: USB Serial support registered for novatel_gps
[    1.365345] usbserial: USB Serial support registered for hp4x
[    1.371110] usbserial: USB Serial support registered for suunto
[    1.377046] usbserial: USB Serial support registered for siemens_mpi
[    1.383432] usbcore: registered new interface driver usb_ehset_test
[    1.392917] input: 30370000.snvs:snvs-powerkey as /devices/platform/soc@0/30000000.bus/30370000.snvs/30370000.snvs:snvs-powerkey/input/input0
[    1.407238] i2c /dev entries driver
[    1.412861] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    1.421904] imx2-wdt 30280000.watchdog: timeout 60 sec (nowayout=0)
[    1.428589] Bluetooth: HCI UART driver ver 2.3
[    1.433050] Bluetooth: HCI UART protocol H4 registered
[    1.438198] Bluetooth: HCI UART protocol BCSP registered
[    1.443533] Bluetooth: HCI UART protocol LL registered
[    1.448677] Bluetooth: HCI UART protocol ATH3K registered
[    1.454093] Bluetooth: HCI UART protocol Three-wire (H5) registered
[    1.460475] Bluetooth: HCI UART protocol Broadcom registered
[    1.466155] Bluetooth: HCI UART protocol QCA registered
[    1.471477] EDAC MC: ECC not enabled
[    1.477035] sdhci: Secure Digital Host Controller Interface driver
[    1.483235] sdhci: Copyright(c) Pierre Ossman
[    1.487822] Synopsys Designware Multimedia Card Interface Driver
[    1.494548] sdhci-pltfm: SDHCI platform and OF driver helper
[    1.500956] mmc0: CQHCI version 5.10
[    1.505209] mmc1: CQHCI version 5.10
[    1.509219] mmc2: CQHCI version 5.10
[    1.544008] mmc2: SDHCI controller on 30b60000.mmc [30b60000.mmc] using ADMA
[    1.553077] ledtrig-cpu: registered to indicate activity on CPUs
[    1.560597] caam 30900000.crypto: device ID = 0x0a16040100000100 (Era 9)
[    1.567407] caam 30900000.crypto: job rings = 3, qi = 0
[    1.584180] caam algorithms registered in /proc/crypto
[    1.590167] caam 30900000.crypto: caam pkc algorithms registered in /proc/crypto
[    1.597576] caam 30900000.crypto: registering rng-caam
[    1.602961] Device caam-keygen registered
[    1.607757] caam_jr 30903000.jr: failed to flush job ring 2
[    1.613417] caam_jr: probe of 30903000.jr failed with error -5
[    1.620039] fsl-jr-uio 30903000.jr: UIO device full name fsl-jr0 initialized
[    1.627487] caam-snvs 30370000.caam-snvs: violation handlers armed - non-secure state
[    1.635972] usbcore: registered new interface driver usbhid
[    1.639878] random: crng init done
[    1.641550] usbhid: USB HID core driver
[    1.649837] mxc-isi 32e02000.isi: mxc_isi.1 registered successfully
[    1.657424] mmc2: Command Queue Engine enabled
[    1.657434] mxc-mipi-csi2-sam 32e40000.csi: 32e40000.csi supply mipi-phy not found, using dummy regulator
[    1.661951] mmc2: new HS400 Enhanced strobe MMC card at address 0001
[    1.671901] mxc-mipi-csi2-sam 32e40000.csi: lanes: 4, hs_settle: 16, clk_settle: 0, wclk: 0, freq: 500000000
[    1.678410] mmcblk2: mmc2:0001 DG4016 14.7 GiB 
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x10, arg2:0x1
[    1.705121] mxc-mipi-csi2-sam 32e50000.csi: 32e50000.csi supply mipi-phy not found, using dummy regulator
[    1.705463] mmcblk2boot0: mmc2:0001 DG4016 partition 1 4.00 MiB
[    1.715154] mxc-mipi-csi2-sam 32e50000.csi: lanes: 2, hs_settle: 13, clk_settle: 2, wclk: 1, freq: 266000000
[    1.720910] mmcblk2boot1: mmc2:0001 DG4016 partition 2 4.00 MiB
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x10, arg2:0x0
[    1.749694] mmcblk2rpmb: mmc2:0001 DG4016 partition 3 4.00 MiB, chardev (237:0)
[    1.757933]  mmcblk2: p1 p2
[    1.758120] No fsl,qman node
[    1.763640] Freescale USDPAA process driver
[    1.767826] fsl-usdpaa: no region found
[    1.771675] Freescale USDPAA process IRQ driver
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x6, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x7, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x4, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x4, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x7, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x6, arg2:0x0
[    1.885343] Galcore version 6.4.3.p1.305572
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x6, arg2:0x1
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x1
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x4, arg2:0x1
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x7, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x4, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x7, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x6, arg2:0x0
[    2.108642] [drm] Initialized vivante 1.0.0 20170808 for 40000000.mix_gpu_ml on minor 0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x9, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0xa, arg2:0x1
[    2.143474] hantrodec 0 : module inserted. Major = 236
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0xa, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x9, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x9, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0xb, arg2:0x1
[    2.200578] hantrodec 1 : module inserted. Major = 236
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0xb, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x9, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x9, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0xc, arg2:0x1
[    2.258025] hantroenc: HW at base <0000000038320000> with ID <0x80006200>
[    2.265036] hx280enc: module inserted. Major <235>
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0xc, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x9, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[    2.519093] imx-cdnhdmi sound-hdmi: ASoC: failed to init link imx8 hdmi: -517
[    2.526274] imx-cdnhdmi sound-hdmi: snd_soc_register_card failed (-517)
[    2.533181] debugfs: Directory '30cc0000.xcvr' with parent 'imx-audio-xcvr' already present!
[    2.541713] imx-xcvr sound-xcvr: snd-soc-dummy-dai <-> 30cc0000.xcvr mapping ok
[    2.549032] imx-xcvr sound-xcvr: ASoC: no DMI vendor name!
[    2.555221] pktgen: Packet Generator for packet performance testing. Version: 2.75
[    2.563061] NET: Registered protocol family 26
[    2.567997] NET: Registered protocol family 10
[    2.572945] Segment Routing with IPv6
[    2.576660] NET: Registered protocol family 17
[    2.581135] bridge: filtering via arp/ip/ip6tables is no longer available by default. Update your scripts to load br_netfilter if you need this.
[    2.594175] Bluetooth: RFCOMM TTY layer initialized
[    2.599065] Bluetooth: RFCOMM socket layer initialized
[    2.604224] Bluetooth: RFCOMM ver 1.11
[    2.607983] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[    2.613298] Bluetooth: BNEP filters: protocol multicast
[    2.618530] Bluetooth: BNEP socket layer initialized
[    2.623500] Bluetooth: HIDP (Human Interface Emulation) ver 1.2
[    2.629425] Bluetooth: HIDP socket layer initialized
[    2.634418] 8021q: 802.1Q VLAN Support v1.8
[    2.638617] lib80211: common routines for IEEE802.11 drivers
[    2.644383] 9pnet: Installing 9P2000 support
[    2.648681] tsn generic netlink module v1 init...
[    2.653445] Key type dns_resolver registered
[    2.658008] registered taskstats version 1
[    2.662120] Loading compiled-in X.509 certificates
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[    2.710842] imx-rpmsg 55000000.rpmsg: assigned reserved memory node vdevbuffer@55400000
[    2.739411] virtio_rpmsg_bus virtio0: rpmsg host is online
[    2.752139] pca9450 0-0025: Device ID=0x31
[    2.756250] pca9450 0-0025: gpio_intr = 3
[    2.760266] pca9450 0-0025: chip_irq=85
[    2.792163] i2c i2c-0: IMX I2C adapter registered
[    2.887703] ov5645 1-003c: ov5645_write_reg: write reg error -6: reg=3103, val=11
[    2.895199] ov5645 1-003c: could not set init registers
[    2.900433] ov5645 1-003c: could not power up OV5645
[    2.905632] i2c i2c-1: IMX I2C adapter registered
[    2.918034] rtc rtc0: invalid alarm value: 2024-09-11T30:48:00
[    2.924001] rtc-pcf8563 2-0051: registered as rtc0
[    2.930796] i2c i2c-2: IMX I2C adapter registered
[    2.936326] i2c i2c-3: IMX I2C adapter registered
[    2.941804] imx8mq-usb-phy 381f0040.usb-phy: 381f0040.usb-phy supply vbus not found, using dummy regulator
[    2.951678] imx8mq-usb-phy 382f0040.usb-phy: 382f0040.usb-phy supply vbus not found, using dummy regulator
[    2.963232] pwm-backlight lvds_backlight: lvds_backlight supply power not found, using dummy regulator
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x1, arg2:0x1
[    2.985506] imx6q-pcie 33800000.pcie: 33800000.pcie supply epdev_on not found, using dummy regulator
[    2.985761] pwm-backlight dsi_backlight: dsi_backlight supply power not found, using dummy regulator
[    2.994880] imx6q-pcie 33800000.pcie: EXT REF_CLK is used!.
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[    3.022910] imx6q-pcie 33800000.pcie: PCIe PHY PLL clock is locked.
[    3.026064] [drm] Supports vblank timestamp caching Rev 2 (21.10.2013).
[    3.035817] [drm] No driver support for vblank timestamp query.
[    3.041925] imx-drm display-subsystem: bound imx-lcdifv3-crtc.0 (ops lcdifv3_crtc_ops)
[    3.049989] imx-drm display-subsystem: bound imx-lcdifv3-crtc.1 (ops lcdifv3_crtc_ops)
[    3.055471] imx6q-pcie 33800000.pcie: PCIe PLL locked after 0 us.
[    3.057949] imx-drm display-subsystem: bound imx-lcdifv3-crtc.2 (ops lcdifv3_crtc_ops)
[    3.064042] imx6q-pcie 33800000.pcie: host bridge /pcie@33800000 ranges:
[    3.071988] imx_sec_dsim_drv 32e60000.mipi_dsi: version number is 0x1060200
[    3.078624] imx6q-pcie 33800000.pcie:   No bus range found for /pcie@33800000, using [bus 00-ff]
[    3.078639] imx6q-pcie 33800000.pcie:    IO 0x1ff80000..0x1ff8ffff -> 0x00000000
[    3.085869] imx-drm display-subsystem: bound 32e60000.mipi_dsi (ops imx_sec_dsim_ops)
[    3.094382] imx6q-pcie 33800000.pcie:   MEM 0x18000000..0x1fefffff -> 0x18000000
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[    3.132000] spi_imx 30830000.spi: probed
[    3.136905] pps pps0: new PPS source ptp0
[    3.147641] libphy: fec_enet_mii_bus: probed
[    3.153350] fec 30be0000.ethernet eth0: registered PHC device 0
[    3.159939] imx-dwmac 30bf0000.ethernet: IRQ eth_lpi not found
[    3.165893] imx-dwmac 30bf0000.ethernet: no reset control found
[    3.171978] imx-dwmac 30bf0000.ethernet: User ID: 0x10, Synopsys ID: 0x51
[    3.178781] imx-dwmac 30bf0000.ethernet: 	DWMAC4/5
[    3.183580] imx-dwmac 30bf0000.ethernet: DMA HW capability register supported
[    3.190721] imx-dwmac 30bf0000.ethernet: RX Checksum Offload Engine supported
[    3.197862] imx-dwmac 30bf0000.ethernet: TX Checksum insertion supported
[    3.204568] imx-dwmac 30bf0000.ethernet: Wake-Up On Lan supported
[    3.210686] imx-dwmac 30bf0000.ethernet: Enable RX Mitigation via HW Watchdog Timer
[    3.218353] imx-dwmac 30bf0000.ethernet: device MAC address 26:f8:2d:f4:43:e5
[    3.225507] imx-dwmac 30bf0000.ethernet: Enabled Flow TC (entries=8)
[    3.231882] imx-dwmac 30bf0000.ethernet: Enabling HW TC (entries=256, max_off=256)
[    3.239553] libphy: stmmac: probed
[    3.245585] OF: graph: no port node found in /usb-phy@381f0040
[    3.252203] xhci-hcd xhci-hcd.0.auto: xHCI Host Controller
[    3.257723] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 1
[    3.265723] xhci-hcd xhci-hcd.0.auto: hcc params 0x0220fe6c hci version 0x110 quirks 0x0000002001810010
[    3.275269] xhci-hcd xhci-hcd.0.auto: irq 78, io mem 0x38200000
[    3.281750] hub 1-0:1.0: USB hub found
[    3.285539] hub 1-0:1.0: 1 port detected
[    3.289671] xhci-hcd xhci-hcd.0.auto: xHCI Host Controller
[    3.295172] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 2
[    3.302842] xhci-hcd xhci-hcd.0.auto: Host supports USB 3.0 SuperSpeed
[    3.309415] usb usb2: We don't know the algorithms for LPM for this host, disabling LPM.
[    3.317856] hub 2-0:1.0: USB hub found
[    3.321632] hub 2-0:1.0: 1 port detected
[    3.326865] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    3.335215] imx-cpufreq-dt imx-cpufreq-dt: cpu speed grade 7 mkt segment 2 supported-hw 0x80 0x4
[    3.347084] mmc0: CQHCI version 5.10
[    3.350722] sdhci-esdhc-imx 30b40000.mmc: allocated mmc-pwrseq
[    3.387689] mmc0: SDHCI controller on 30b40000.mmc [30b40000.mmc] using ADMA
[    3.395922] imx8mp-pinctrl 30330000.pinctrl: pin MX8MP_IOMUXC_NAND_WE_B already requested by 30b60000.mmc; cannot claim for 30b50000.mmc
[    3.408191] imx8mp-pinctrl 30330000.pinctrl: pin-73 (30b50000.mmc) status -22
[    3.415334] imx8mp-pinctrl 30330000.pinctrl: could not request pin 73 (MX8MP_IOMUXC_NAND_WE_B) from group usdhc3grp  on device 30330000.pinctrl
[    3.428219] sdhci-esdhc-imx 30b50000.mmc: Error applying setting, reverse things back
[    3.436107] sdhci-esdhc-imx: probe of 30b50000.mmc failed with error -22
[    3.444482] debugfs: Directory '30c50000.sai' with parent 'bt-sco-audio' already present!
[    3.452758] asoc-simple-card sound-bt-sco: bt-sco-pcm-wb <-> 30c50000.sai mapping ok
[    3.460513] asoc-simple-card sound-bt-sco: ASoC: no DMI vendor name!
[    3.466915] debugfs: File 'Playback' in directory 'dapm' already present!
[    3.473718] debugfs: File 'Capture' in directory 'dapm' already present!
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x5, arg2:0x0
[    3.521354] debugfs: Directory '30c30000.sai' with parent 'nau8822-audio' already present!
[    3.529799] debugfs: Directory '30c90000.easrc' with parent 'nau8822-audio' already present!
[    3.538331] imx-nau8822 sound-nau8822: nau8822-hifi <-> 30c30000.sai mapping ok
[    3.548121] imx-nau8822 sound-nau8822: snd-soc-dummy-dai <-> 30c90000.easrc mapping ok
[    3.556170] imx-nau8822 sound-nau8822: nau8822-hifi <-> 30c30000.sai mapping ok
[    3.563547] imx-nau8822 sound-nau8822: ASoC: no DMI vendor name!
[    3.600521] mmc0: new ultra high speed SDR104 SDIO card at address 0001
[    3.639412] usb 1-1: new high-speed USB device number 2 using xhci-hcd
[    3.681440] imx-cdnhdmi sound-hdmi: ASoC: failed to init link imx8 hdmi: -517
[    3.688614] imx-cdnhdmi sound-hdmi: snd_soc_register_card failed (-517)
[    3.710387] [drm] Supports vblank timestamp caching Rev 2 (21.10.2013).
[    3.717026] [drm] No driver support for vblank timestamp query.
[    3.723137] imx-drm display-subsystem: bound imx-lcdifv3-crtc.0 (ops lcdifv3_crtc_ops)
[    3.731168] imx-drm display-subsystem: bound imx-lcdifv3-crtc.1 (ops lcdifv3_crtc_ops)
[    3.739113] imx-drm display-subsystem: bound imx-lcdifv3-crtc.2 (ops lcdifv3_crtc_ops)
[    3.747108] imx_sec_dsim_drv 32e60000.mipi_dsi: version number is 0x1060200
[    3.754323] imx-drm display-subsystem: bound 32e60000.mipi_dsi (ops imx_sec_dsim_ops)
[    3.762218] imx-drm display-subsystem: bound 32c00000.bus:ldb@32ec005c (ops imx8mp_ldb_ops)
[    3.770761] dwhdmi-imx 32fd8000.hdmi: Detected HDMI TX controller v2.13a with HDCP (samsung_dw_hdmi_phy2)
[    3.780717] dwhdmi-imx 32fd8000.hdmi: registered DesignWare HDMI I2C bus driver
[    3.788842] imx-drm display-subsystem: bound 32fd8000.hdmi (ops dw_hdmi_imx_ops)
[    3.796528] [drm] Initialized imx-drm 1.0.0 20120507 for display-subsystem on minor 1
[    3.853496] hub 1-1:1.0: USB hub found
[    3.857303] hub 1-1:1.0: 4 ports detected
[    3.923903] imx-drm display-subsystem: fb0: imx-drmdrmfb frame buffer device
[    3.931626] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    3.935476] usb 2-1: new SuperSpeed Gen 1 USB device number 2 using xhci-hcd
[    3.940463] debugfs: Directory '30cb0000.aud2htx' with parent 'audio-hdmi' already present!
[    3.955054] imx-cdnhdmi sound-hdmi: i2s-hifi <-> 30cb0000.aud2htx mapping ok
[    3.962135] imx-cdnhdmi sound-hdmi: ASoC: no DMI vendor name!
[    3.968151] input: audio-hdmi HDMI Jack as /devices/platform/sound-hdmi/sound/card3/input2
[    3.977080] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    3.981515] hub 2-1:1.0: USB hub found
[    3.989250] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    3.997186] hub 2-1:1.0: 4 ports detected
[    4.002463] input: keys as /devices/platform/keys/input/input3
[    4.004467] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    4.016917] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    4.025292] rtc-pcf8563 2-0051: setting system clock to 2024-09-16T09:23:21 UTC (1726478601)
[    4.034167] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[    4.043657] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[    4.050240] platform regulatory.0: Direct firmware load for regulatory.db failed with error -2
[    4.051409] clk: Not disabling unused clocks
[    4.058866] platform regulatory.0: Falling back to sysfs fallback for: regulatory.db
[    4.063170] ALSA device list:
[    4.073894]   #0: imx-audio-xcvr
[    4.077133]   #1: bt-sco-audio
[    4.080194]   #2: nau8822-audio
[    4.083334]   #3: audio-hdmi
[    4.087411] imx6q-pcie 33800000.pcie: Phy link never came up
[    4.093136] imx6q-pcie 33800000.pcie: failed to initialize host
[    4.099058] imx6q-pcie 33800000.pcie: unable to add pcie port.
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x1, arg2:0x0
[    4.132509] EXT4-fs (mmcblk2p2): recovery complete
[    4.137849] EXT4-fs (mmcblk2p2): mounted filesystem with ordered data mode. Opts: (null)
[    4.145986] VFS: Mounted root (ext4 filesystem) on device 179:2.
[    4.156329] devtmpfs: mounted
[    4.160002] Freeing unused kernel memory: 2880K
[    4.164607] Run /sbin/init as init process
[    4.211443] usb 1-1.4: new full-speed USB device number 3 using xhci-hcd
[    4.241001] systemd[1]: systemd 243.2+ running in system mode. (+PAM -AUDIT -SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP -LIBCRYPTSETUP -GCRYPT -GNUTLS +ACL +XZ -LZ4 -SECCOMP +BLKID -ELFUTILS +KMOD -IDN2 -IDN -PCRE2 default-hierarchy=hybrid)
[    4.262899] systemd[1]: Detected architecture arm64.

Welcome to NXP i.MX Release Distro 5.4-zeus (zeus)!

[    4.315796] systemd[1]: Set hostname to <OK8MP>.
[    4.446358] systemd[1]: /lib/systemd/system/dbus.socket:5: ListenStream= references a path below legacy directory /var/run/, updating /var/run/dbus/system_bus_socket → /run/dbus/system_bus_socket; please update the unit file accordingly.
[    4.469406] i.mx8mm_thermal 30260000.tmu: failed to register thermal zone sensor[0]: 0
[    4.480588] systemd[1]: /lib/systemd/system/syslogd.service:8: PIDFile= references a path below legacy directory /var/run/, updating /var/run/syslogd.pid → /run/syslogd.pid; please update the unit file accordingly.
[    4.504407] systemd[1]: /lib/systemd/system/rpcbind.socket:5: ListenStream= references a path below legacy directory /var/run/, updating /var/run/rpcbind.sock → /run/rpcbind.sock; please update the unit file accordingly.
[    4.527040] systemd[1]: /lib/systemd/system/lighttpd.service:7: PIDFile= references a path below legacy directory /var/run/, updating /var/run/lighttpd.pid → /run/lighttpd.pid; please update the unit file accordingly.
[    4.549817] systemd[1]: /lib/systemd/system/klogd.service:8: PIDFile= references a path below legacy directory /var/run/, updating /var/run/klogd.pid → /run/klogd.pid; please update the unit file accordingly.
[    4.629272] systemd[1]: system-getty.slice: unit configures an IP firewall, but the local system does not support BPF/cgroup firewalling.
[    4.641662] systemd[1]: (This warning is only shown for the first unit using IP firewalling.)
[  OK  ] Created slice system-getty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[  OK  ] Created slice User and Session Slice.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[  OK  ] Reached target Paths.
[  OK  ] Reached target Remote File Systems.
[  OK  ] Reached target Slices.
[  OK  ] Reached target Swap.
[  OK  ] Listening on Syslog Socket.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket.
[  OK  ] Listening on Network Service Netlink Socket.
[  OK  ] Listening on udev Control Socket.
[  OK  ] Listening on udev Kernel Socket.
         Mounting Huge Pages File System...
         Mounting POSIX Message Queue File System...
         Mounting Kernel Debug File System...
         Mounting Temporary Directory (/tmp)...
         Starting Journal Service...
         Mounting Kernel Configuration File System...
         Starting Remount Root and Kernel File Systems...
         Starting Apply Kernel Variables...
         Starting udev Coldplug all Devices...
[  OK  ] Mounted Huge Pages File System.
[    5.078905] EXT4-fs (mmcblk2p2): re-mounted. Opts: (null)
[  OK  ] Started Journal Service.
[  OK  ] Mounted POSIX Message Queue File System.
[  OK  ] Mounted Kernel Debug File System.
[  OK  ] Mounted Temporary Directory (/tmp).
[  OK  ] Mounted Kernel Configuration File System.
[  OK  ] Started Remount Root and Kernel File Systems.
[  OK  ] Started Apply Kernel Variables.
         Starting Flush Journal to Persistent Storage...
[    5.212984] systemd-journald[241]: Received client request to flush runtime journal.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Started Flush Journal to Persistent Storage.
[  OK  ] Started Create Static Device Nodes in /dev.
[  OK  ] Reached target Local File Systems (Pre).
         Mounting /var/volatile...
         Starting udev Kernel Device Manager...
[  OK  ] Mounted /var/volatile.
         Starting Load/Save Random Seed...
[  OK  ] Reached target Local File Systems.
         Starting Create Volatile Files and Directories...
[  OK  ] Started Load/Save Random Seed.
[  OK  ] Started Create Volatile Files and Directories.
[  OK  ] Started udev Kernel Device Manager.
         Starting Network Time Synchronization...
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started udev Coldplug all Devices.
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target System Time Set.
[  OK  ] Reached target System Time Synchronized.
[  OK  ] Started Daily apt download activities.
[  OK  ] Started Daily rotation of log files.
[  OK  ] Reached target Timers.
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Listening on dropbear.socket.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
[  OK  ] Started Job spooling tools.
[  OK  ] Started autorun.
         Starting Console System Startup Logging...
[  OK  ] Started Periodic Command Scheduler.
[  OK  ] Started D-Bus System Message Bus.
[  OK  ] Started Configuration for i.MX GPU (Former rc_gpu.S).
[  OK  ] Started ISP i.MX 8Mplus daemon.
         Starting Packet Filtering Framework...
         Starting Lighttpd Daemon...
         Starting Telephony service...
         Starting RPC Bind Service...
         Starting System Logging Service...
         Starting Login Service...
[  OK  ] Started TEE Supplicant.
[  OK  ] Started uartsdio8987.
[  OK  ] Started Console System Startup Logging.
[  OK  ] Started Packet Filtering Framework.
[  OK  ] Started Lighttpd Daemon.
[  OK  ] Started RPC Bind Service.
[  OK  ] Started System Logging Service.
[  OK  ] Started Telephony service.
[  OK  ] Created slice system-systemd\x2dbacklight.slice.
[  OK  ] Created slice system-weston.slice.
[  OK  ] Reached target Network (Pre).
         Starting Kernel Logging Service...
[  OK  ] Started matrix-browser.
         Starting Load/Save Screen … of backlight:dsi_backlight...
         Starting Load/Save Screen …of backlight:lvds_backlight...
         Starting Network Service...
[  OK  ] Started Load/Save Screen B…ss of backlight:dsi_backlight.
[  OK  ] Started Load/Save Screen B…s of backlight:lvds_backlight.
[  OK  ] Started Login Service.
[  OK  ] Started Network Service.
         Starting Network Name Resolution...
[  OK  ] Started Network Name Resolution.
[  OK  ] Reached target Network.
[  OK  ] Reached target Host and Network Name Lookups.
         Starting Avahi mDNS/DNS-SD Stack...
[  OK  ] Started NFS status monitor for NFSv2/3 locking..
         Starting /etc/rc.local Compatibility...
         Starting Permit User Sessions...
[  OK  ] Started Vsftpd ftp daemon.
[  OK  ] Started Kernel Logging Service.
[  OK  ] Started     6.895485] audit: type=1701 audit(1726478604.364:2): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=333 comm="matrix-browser" exe="/usr/bin/forlinx/matrix-browser" sig=6 res=1
;39m/etc/rc.local Compatibility.
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttymxc1.
[    6.999811] YT8521 ethernet 30be0000.ethernet-1:01: attached PHY driver [YT8521 ethernet] (mii_bus:phy_addr=30be0000.ethernet-1:01, irq=POLL)
[    7.014869] imx-sdma 30bd0000.dma-controller: loaded firmware 4.5
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
         Starting Update UTMP about System Runlevel Changes...
[    7.068631] imx-dwmac 30bf0000.ethernet eth1: PHY [stmmac-1:01] driver [YT8521 ethernet]
         Starting Weston Wayland Compositor (on tty7)...
[    7.111433] imx-dwmac 30bf0000.ethernet eth1: No Safety Features support found
[  OK  ] Started Weston Wayland Compositor (on tty7).
[    7.130020] imx-dwmac 30bf0000.ethernet eth1: IEEE 1588-2008 Advanced Timestamp supported
[    7.144150] imx-dwmac 30bf0000.ethernet eth1: registered PTP clock
[    7.154648] imx-dwmac 30bf0000.ethernet eth1: configuring for phy/rgmii-id link mode
[    7.172914] 8021q: adding VLAN 0 to HW filter on device eth1
[  OK  ] Started Update UTMP about System Runlevel Changes.
[  OK  ] Created slice User Slice of UID 0.
         Starting Save/Restore Sound Card State...
         Starting User Runtime Directory /run/user/0...
[  OK  ] Started User Runtime Directory /run/user/0.
         Starting User Manager for UID 0...
[    7.405754] audit: type=1006 audit(1726478604.876:3): pid=458 uid=0 old-auid=4294967295 auid=0 tty=(none) old-ses=4294967295 ses=1 res=1
[  OK  ] Started User Manager for UID 0.
[  OK  ] Started Session c1 of user root.
[  OK  ] Stopped matrix-browser.
[  OK  ] Started matrix-browser.
[  OK  ] Created slice system-systemd\x2dfsck.slice.
[  OK  ] Found device /dev/mmcblk2p1.
         Starting File System Check on /dev/mmcblk2p1...
[  OK  ] Started File System Check on /dev/mmcblk2p1.
         Mounting /run/media/mmcblk2p1...
[  OK  ] Mounted /run/media/mmcblk2p1.
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x6, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x1
[INFO  1] (hvisor::arch::aarch64::trap:270) SMC from CPU1, func_id:0xc2000000, arg0:0x3, arg1:0x7, arg2:0x1
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x1
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x8, arg2:0x0
[  OK  ] Started Save/Restore Sound Card State.
[  OK  ] Reached target Sound Card.
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x7, arg2:0x0
[INFO  0] (hvisor::arch::aarch64::trap:270) SMC from CPU0, func_id:0xc2000000, arg0:0x3, arg1:0x6, arg2:0x0
[   11.099840] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
[   11.106822] fec 30be0000.ethernet eth0: Link is Up - 1Gbps/Full - flow control rx/tx

NXP i.MX Release Distro 5.4-zeus OK8MP ttymxc1

```

