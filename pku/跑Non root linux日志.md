# 跑Non root linux磁盘日志：

## 第一次

```
[    0.000000] Linux version 5.4.0-dirty (korwylee@LAPTOP-VRNM9I0I) (gcc version 10.3.1 20210621 (GNU Toolchain for the A-profile Architecture 10.3-2021.07 (arm-10.29))) #2 SMP PREEMPT Wed Oct 11 14:42:05 CST 2023
[    0.000000] Machine model: Jailhouse cell on QEMU ARM64
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 32 MiB at 0x0000000076000000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000070000000-0x0000000077ffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x75fba800-0x75fbbfff]
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000070000000-0x0000000077ffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000070000000-0x0000000077ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000070000000-0x0000000077ffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.1
[    0.000000] percpu: Embedded 22 pages/cpu s52952 r8192 d28968 u90112
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: ARM erratum 832075
[    0.000000] CPU features: detected: GIC system register CPU interface
[    0.000000] CPU features: detected: ARM erratum 834220
[    0.000000] CPU features: detected: EL2 vector hardening
[    0.000000] ARM_SMCCC_ARCH_WORKAROUND_1 missing from firmware
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32256
[    0.000000] Policy zone: DMA32
[    0.000000] Kernel command line: console=ttyAMA0,115200 root=/dev/vda rdinit=/linuxrc
[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 54220K/131072K available (12092K kernel code, 1862K rwdata, 6388K rodata, 5056K init, 448K bss, 44084K reserved, 32768K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
[    0.000000]  Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: 224 SPIs implemented
[    0.000000] GICv3: 0 Extended SPIs implemented
[    0.000000] GICv3: Distributor has no Range Selector support
[    0.000000] GICv3: 16 PPIs implemented
[    0.000000] GICv3: no VLPI support, no direct LPI support
[    0.000000] GICv3: CPU0: found redistributor 2 region 0:0x00000000080e0000
[    0.000000] ITS: No ITS available, not enabling LPIs
[    0.000000] random: get_random_bytes called from start_kernel+0x2b4/0x448 with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000155] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.014214] Console: colour dummy device 80x25
[    0.018321] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.018553] pid_max: default: 32768 minimum: 301
[    0.020346] LSM: Security Framework initializing
[    0.022689] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.022746] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.111732] ASID allocator initialised with 32768 entries
[    0.118909] rcu: Hierarchical SRCU implementation.
[    0.133156] EFI services will not be available.
[    0.135018] smp: Bringing up secondary CPUs ...
[    0.175428] Detected PIPT I-cache on CPU1
[    0.176393] GICv3: CPU1: found redistributor 3 region 0:0x0000000008100000
[    0.176928] CPU1: Booted secondary processor 0x0000000003 [0x411fd070]
[    0.182727] smp: Brought up 1 node, 2 CPUs
[    0.182847] SMP: Total of 2 processors activated.
[    0.182996] CPU features: detected: 32-bit EL0 Support
[    0.183142] CPU features: detected: CRC32 instructions
[    2.781053] CPU: All CPU(s) started at EL1
[    2.781619] alternatives: patching kernel code
[    2.828559] devtmpfs: initialized
[    2.849794] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    2.850171] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    2.859740] pinctrl core: initialized pinctrl subsystem
[    2.880915] DMI not present or invalid.
[    2.892150] NET: Registered protocol family 16
[    2.916222] DMA: preallocated 256 KiB pool for atomic allocations
[    2.916453] audit: initializing netlink subsys (disabled)
[    2.921272] audit: type=2000 audit(2.620:1): state=initialized audit_enabled=0 res=1
[    2.927475] cpuidle: using governor menu
[    2.929400] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    2.935922] Serial: AMBA PL011 UART driver
[    2.952509] 9000000.serial: ttyAMA0 at MMIO 0x9000000 (irq = 5, base_baud = 0) is a PL011 rev1
[    2.976714] printk: console [ttyAMA0] enabled
[    3.023878] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    3.024142] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    3.024389] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    3.024832] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    3.076413] cryptd: max_cpu_qlen set to 1000
[    3.131611] ACPI: Interpreter disabled.
[    3.137473] iommu: Default domain type: Translated
[    3.140407] vgaarb: loaded
[    3.143678] SCSI subsystem initialized
[    3.149601] usbcore: registered new interface driver usbfs
[    3.150412] usbcore: registered new interface driver hub
[    3.151057] usbcore: registered new device driver usb
[    3.153080] pps_core: LinuxPPS API ver. 1 registered
[    3.153276] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    3.155476] PTP clock support registered
[    3.156393] EDAC MC: Ver: 3.0.0
[    3.163449] FPGA manager framework
[    3.165342] Advanced Linux Sound Architecture Driver Initialized.
[    3.186834] clocksource: Switched to clocksource arch_sys_counter
[    3.189333] VFS: Disk quotas dquot_6.6.0
[    3.192670] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    3.197036] pnp: PnP ACPI: disabled
[    3.281702] thermal_sys: Registered thermal governor 'step_wise'
[    3.281832] thermal_sys: Registered thermal governor 'power_allocator'
[    3.284835] NET: Registered protocol family 2
[    3.295929] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    3.296315] TCP established hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    3.296717] TCP bind hash table entries: 1024 (order: 2, 16384 bytes, linear)
[    3.297181] TCP: Hash tables configured (established 1024 bind 1024)
[    3.300173] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    3.300672] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    3.304216] NET: Registered protocol family 1
[    3.313581] RPC: Registered named UNIX socket transport module.
[    3.313862] RPC: Registered udp transport module.
[    3.313996] RPC: Registered tcp transport module.
[    3.314154] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    3.314489] PCI: CLS 0 bytes, default 64
[    3.321698] Trying to unpack rootfs image as initramfs...
[    3.476514] Freeing initrd memory: 1096K
[    3.479735] kvm [1]: HYP mode not available
[    3.897236] Initialise system trusted keyrings
[    3.901651] workingset: timestamp_bits=44 max_order=15 bucket_order=0
[    3.920331] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    3.927529] NFS: Registering the id_resolver key type
[    3.928131] Key type id_resolver registered
[    3.928302] Key type id_legacy registered
[    3.928932] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    3.931772] 9p: Installing v9fs 9p2000 file system support
[    3.959916] Key type asymmetric registered
[    3.960443] Asymmetric key parser 'x509' registered
[    3.961206] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 245)
[    3.961623] io scheduler mq-deadline registered
[    3.961820] io scheduler kyber registered
[    3.980361] pci-host-generic 7000000.pci: host bridge /pci@7000000 ranges:
[    3.981869] pci-host-generic 7000000.pci:   MEM 0x10000000..0x1000ffff -> 0x10000000
[    3.984836] pci-host-generic 7000000.pci: ECAM at [mem 0x07000000-0x070fffff] for [bus 00]
[    3.987621] pci-host-generic 7000000.pci: PCI host bridge to bus 0000:00
[    3.988557] pci_bus 0000:00: root bus resource [bus 00]
[    3.988848] pci_bus 0000:00: root bus resource [mem 0x10000000-0x1000ffff]
[    3.992680] pci 0000:00:00.0: [1af4:1110] type 00 class 0xff0100
[    3.995126] pci 0000:00:00.0: reg 0x10: [mem 0x00000000-0x000000ff 64bit]
[    4.006093] pci 0000:00:00.0: BAR 0: assigned [mem 0x10000000-0x100000ff 64bit]
[    4.015595] EINJ: ACPI disabled.
Unhandled data read at 0xa003e00(4)

FATAL: unhandled trap (exception class 0x24)
Cell state before exception:
 pc: ffff800010627164   lr: ffff80001062715c spsr: 60000005     EL1
 sp: ffff80001002bbb0  esr: 24 1 1804006
 x0: ffff800010025e00   x1: ffff000034848000   x2: 0000000000000000
 x3: ffff000034a7a400   x4: ffff000034957610   x5: ffff800010025000
 x6: 0040000000000001   x7: 0140000000000000   x8: ffff80000fffffff
 x9: 0000000000000000  x10: ffff8000117a28d0  x11: ffff800010027000
x12: 0000000000000030  x13: ffffffffffffffff  x14: ffff0000349c688a
x15: 000000000a003000  x16: 0000000000009943  x17: 0000000000000001
x18: ffff8000111d5490  x19: ffff000034a7a480  x20: ffff000034957410
x21: ffff00003497dd00  x22: ffff000034957400  x23: ffff800011904770
x24: ffff8000118ed6e0  x25: 0000000000000000  x26: ffff8000112a0520
x27: 0000000000000000  x28: 0000000000000000  x29: ffff80001002bbb0

Parking CPU 2 (Cell: "qemu-arm64-linux-demo")
```

命令：

```
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/vda rdinit=/linuxrc" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb -i ../initramfs.cpio.gz
```

## 第二次

命令：

```
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
    -drive if=none,file=second-ubuntu-20.04-rootfs_ext4.img,id=hd1,format=raw \
    -device virtio-blk-device,drive=hd1 \
    -drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01 \
    -machine dumpdtb=qemu-virt.dtb
```



```
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/vda rw" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb
```

报错信息：

```
arm64@kernel-54:~/jail/jailhouse/tools$ sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/vda rw" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb
text_offset is 524288
Adding virtual PCI device 00:00.0 to cell "qemu-arm64-linux-demo"
Unhandled data read at 0x8080008(8)

FATAL: unhandled trap (exception class 0x24)
Cell state before exception:
 pc: ffffadf310089c58   lr: ffffadf31008a8b8 spsr: 800001c5     EL1
 sp: ffff800010123e80  esr: 24 1 1c1c006
 x0: ffffadf310fe2018   x1: ffff8000100a0008   x2: ffff800010123e80
 x3: ffff520d1bee8000   x4: 0000000000000000   x5: ffffadf31139d120
 x6: 00000000a0a0a0a0   x7: 0000000000000004   x8: ffffadf311571000
 x9: 0000000000000000  x10: ffffadf311571600  x11: ffffadf3113b2000
x12: ffffadf311571000  x13: 0000000000000000  x14: ffffadf3113b2448
x15: ffffadf311571fb2  x16: 0000000000000000  x17: 0000000000000000
x18: 0000000000000030  x19: ffff00002c40d200  x20: ffff00002c40d200
x21: 0000000000000002  x22: ffffadf3115ab000  x23: 0000000000000001
x24: 0000000000000000  x25: 0000000000000000  x26: 0000000000000000
x27: 0000000000000000  x28: 0000000000000000  x29: ffff800010123e80

Parking CPU 2 (Cell: "qemu-arm64")
[  142.664973] CPU2: failed to come online
[  142.665475] CPU2: failed in unknown state : 0x0
[  142.666369] bad: scheduling from the idle thread!
[  142.670221] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000000
[  142.671967] Mem abort info:
[  142.673133]   ESR = 0x86000006
[  142.673297]   EC = 0x21: IABT (current EL), IL = 32 bits
[  142.674034]   SET = 0, FnV = 0
[  142.674620]   EA = 0, S1PTW = 0
[  142.675392] user pgtable: 4k pages, 48-bit VAs, pgdp=00000000626a5000
[  142.675993] [0000000000000000] pgd=000000006327a003, pud=0000000068af6003, pmd=0000000000000000
[  142.677875] Internal error: Oops: 86000006 [#1] PREEMPT SMP
[  142.679974] Modules linked in: jailhouse(O) crct10dif_ce drm ip_tables x_tables ipv6 nf_defrag_ipv6
[  142.685043] CPU: 10 PID: 510 Comm: python Tainted: G           O      5.4.0-dirty #2
[  142.688293] Hardware name: linux,dummy-virt (DT)
[  142.689845] pstate: 40000085 (nZcv daIf -PAN -UAO)
[  142.691953] pc : 0x0
[  142.692949] lr : do_set_cpus_allowed+0x16c/0x180
[  142.693453] sp : ffff8000105f3ae0
[  142.694087] x29: ffff8000105f3ae0 x28: 0000000000000001
[  142.694962] x27: 0000000000000000 x26: ffffadf31139a118
[  142.697016] x25: ffffadf31139a2b4 x24: 0000000000000001
[  142.698106] x23: ffffadf31139a118 x22: ffffadf31139a138
[  142.700527] x21: ffff00002c632940 x20: ffff00002c632940
[  142.701843] x19: ffff00002ced5d40 x18: 0000000000000020
[  142.702667] x17: 0000000000000000 x16: 0000000000000000
[  142.703389] x15: ffffadf311571fb0 x14: ffffadf3113b2448
[  142.704526] x13: 0000000000000000 x12: ffffadf311571000
[  142.706546] x11: ffffadf3113b2000 x10: ffffadf311571600
[  142.707520] x9 : 0000000000000000 x8 : ffffadf311571000
[  142.708757] x7 : 0000000000000001 x6 : 0000000000000000
[  142.709163] x5 : 0000000000000000 x4 : 0000000000000000
[  142.709404] x3 : 0000000000000000 x2 : 000000000000000a
[  142.709943] x1 : ffff00002c632940 x0 : ffff00002ced5d40
[  142.710558] Call trace:
[  142.712017]  0x0
[  142.713688]  cpuset_cpus_allowed_fallback+0x5c/0x6c
[  142.715452]  select_fallback_rq+0x1c4/0x1e4
[  142.716145]  sched_cpu_dying+0x1c8/0x2f4
[  142.716428]  cpuhp_invoke_callback+0x84/0x1f0
[  142.717210]  _cpu_up+0x18c/0x1b0
[  142.717471]  do_cpu_up+0x74/0xb0
[  142.718533]  cpu_up+0x10/0x20
[  142.720785]  jailhouse_cmd_cell_create+0x3fc/0x4c0 [jailhouse]
[  142.721713]  jailhouse_ioctl+0x40/0xd0 [jailhouse]
[  142.722918]  do_vfs_ioctl+0x76c/0x9c0
[  142.723114]  ksys_ioctl+0x78/0xac
[  142.724443]  __arm64_sys_ioctl+0x1c/0x30
[  142.724746]  el0_svc_common.constprop.0+0x68/0x160
[  142.725537]  el0_svc_handler+0x20/0x80
[  142.726313]  el0_svc+0x8/0xc
[  142.727804] Code: bad PC value
[  142.729975] ---[ end trace d4cc14c49bd015dd ]---
[  142.731036] note: python[510] exited with preempt_count 2
```

- [x] 看懂这个函数

int ivshmem_init(struct cell **cell*, struct pci_device **device*)

## 第三次

```
[    3.552063] pci 0000:00:00.0: reg 0x10: [mem 0x00000000-0x000000ff 64bit]
[    3.562948] pci 0000:00:00.0: BAR 0: assigned [mem 0x10000000-0x100000ff 64bit]
[    3.571459] EINJ: ACPI disabled.
FATAL: Invalid MMIO read, address: a003e00, size: 4
```



```
[    3.565074] pci 0000:00:00.0: [1af4:1110] type 00 class 0xff0100
[    3.568329] pci 0000:00:00.0: reg 0x10: [mem 0x00000000-0x000000ff 64bit]
[    3.582978] pci 0000:00:00.0: BAR 0: assigned [mem 0x10000000-0x100000ff 64bit]
[    3.593157] EINJ: ACPI disabled.
[    3.619406] virtio-pci 0000:00:00.0: enabling device (0000 -> 0002)
[    3.660304] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    3.669913] SuperH (H)SCI(F) driver initialized
[    3.671002] msm_serial: driver initialized
[    3.675523] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    3.743235] brd: module loaded
[    3.784959] loop: module loaded
FATAL: Invalid MMIO read, address: a003f0c, size: 1
```

## 第四次

```
[    3.770942] ALSA device list:
[    3.771183]   No soundcards found.
[    3.780761] uart-pl011 9000000.serial: no DMA platform data
[    3.789899] VFS: Cannot open root device "vda" or unknown-block(0,0): error -6
[    3.790163] Please append a correct "root=" boot option; here are the available partitions:
[    3.792753] 0100            4096 ram0
[    3.792802]  (driver?)
[    3.793107] 0101            4096 ram1
[    3.793114]  (driver?)
[    3.793401] 0102            4096 ram2
[    3.793409]  (driver?)
[    3.793771] 0103            4096 ram3
[    3.793777]  (driver?)
[    3.793919] 0104            4096 ram4
[    3.793923]  (driver?)
[    3.794323] 0105            4096 ram5
[    3.794331]  (driver?)
[    3.794467] 0106            4096 ram6
[    3.794470]  (driver?)
[    3.795588] 0107            4096 ram7
[    3.795601]  (driver?)
[    3.796070] 0108            4096 ram8
[    3.796082]  (driver?)
[    3.796395] 0109            4096 ram9
[    3.796404]  (driver?)
[    3.796626] 010a            4096 ram10
[    3.796632]  (driver?)
[    3.796790] 010b            4096 ram11
[    3.796793]  (driver?)
[    3.797037] 010c            4096 ram12
[    3.797044]  (driver?)
[    3.797217] 010d            4096 ram13
[    3.797222]  (driver?)
[    3.797386] 010e            4096 ram14
[    3.797392]  (driver?)
[    3.797612] 010f            4096 ram15
[    3.797618]  (driver?)
[    3.798219] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    3.798823] CPU: 1 PID: 1 Comm: swapper/0 Not tainted 5.4.0-dirty #2
[    3.798997] Hardware name: Jailhouse cell on QEMU ARM64 (DT)
[    3.799629] Call trace:
[    3.799802]  dump_backtrace+0x0/0x134
[    3.800093]  show_stack+0x14/0x20
[    3.800269]  dump_stack+0xb4/0x100
[    3.800400]  panic+0x158/0x324
[    3.800531]  mount_block_root+0x1cc/0x284
[    3.800629]  mount_root+0x11c/0x150
[    3.800790]  prepare_namespace+0x15c/0x1c0
[    3.800999]  kernel_init_freeable+0x210/0x22c
[    3.801142]  kernel_init+0x10/0x100
[    3.801292]  ret_from_fork+0x10/0x18
[    3.802134] SMP: stopping secondary CPUs
[    3.803013] Kernel Offset: disabled
[    3.803486] CPU features: 0x0002,2000608a
[    3.804051] Memory Limit: none
[    3.805379] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

使用`fdisk -l`命令，发现：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310161653264.png" alt="image-20231016165326017" style="zoom:50%;" />

命令行参数应该是/dev/vdb

root cell启动流程：

```
[    5.377307] EINJ: ACPI disabled.
[    5.455674] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    5.466234] SuperH (H)SCI(F) driver initialized
[    5.467867] msm_serial: driver initialized
[    5.473952] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    5.548475] brd: module loaded
[    5.595758] loop: module loaded
[    5.621580] virtio_blk virtio1: [vda] 16777216 512-byte logical blocks (8.59 GB/8.00 GiB)
[    5.658990] virtio_blk virtio2: [vdb] 131072 512-byte logical blocks (67.1 MB/64.0 MiB)
[    5.688604] libphy: Fixed MDIO Bus: probed
```

而现在的报错：

```
[    3.849675] brd: module loaded
[    3.891605] loop: module loaded
[    3.895149] genirq: Setting trigger mode 1 for irq 6 failed (gic_set_type+0x0/0x164)
[    3.895924] virtio_blk: probe of virtio0 failed with error -22
[    3.911977] libphy: Fixed MDIO Bus: probed
```

## 第5次

为non root linux加了79号中断，在host上执行：

```
./disk_enable_jailhouse.sh
```

在guest上执行：

```
./enable_non_root_linux.sh
```

## 步骤

制作新的镜像：

```
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=8192 oflag=direct
mkfs.ext4 ubuntu-20.04-rootfs_ext4.img
sudo mount -t ext4 ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/
sudo cp /usr/bin/qemu-system-aarch64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
```

启动qemu：

```bash
sudo qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 16 \
    -m 1024 \
    -kernel ./linux/arch/arm64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=third-ubuntu-20.04-rootfs_ext4.img,id=hd1,format=raw \
    -device virtio-blk-device,drive=hd1 \
    -drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

为了找到qemu模拟的block设备，dump出qemu的设备树信息：

```bash
    -machine dumpdtb=qemu-virt.dtb
```

qemu指定的设备在设备树中是倒着来的，找到最后一个virtio-mmio区域：

```
virtio_mmio@a003e00 {
    dma-coherent;
    interrupts = <0x00 0x2f 0x01>;
    reg = <0x00 0xa003e00 0x00 0x200>;
    compatible = "virtio,mmio";
};
```

写入inmate-qemu-arm64.dts中。同时为non-root-linux的cell配置文件增加相应的mem region。由于non-root-linux在probe virtio-blk时，用到了79号中断，因此修改cell：

```c
	.irqchips = {
		/* GIC */ {
			.address = 0x08000000,
			.pin_base = 32, // pin_base 表示从第几号中断开始，pin_bitmap的类型为u32[4]，
			.pin_bitmap = { // 每一个元素表示32个中断，其中位设为1的中断，root cell会取消拥有该中断
				(1 << (33 - 32)),
				(1 << 15),  // 修改这里
				0,
				(1 << (140 - 128))
			},
		},
	},
```

启动non-root-linux：

```bash
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/vda rw" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb
```

