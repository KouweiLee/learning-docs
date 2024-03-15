# virtio-blk问题汇总

## mount的问题

当root通过open打开磁盘镜像后, 再mount, 会成功. non root也不会卡住, 但是non root的所有写操作都不起作用了. 只open不Mount的情况下是正常的.

之前是因为, virtio device在标准输出上输出, 此时如果在终端输入字符会中断输出, 进而中断整个进程. 将其改成nohup, 即可改成守护进程而不是后台任务.  

-r为只读挂载, 但报错:

```
non root: # [   88.045567] EXT4-fs (loop0): write access unavailable, cannot proceed (try mounting with noload)

root: /home/arm64/hvisor/tools/rootfs: cannot mount /dev/loop0 read-only.

```

## 性能测试

### dd方法

#### non root

```
# dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=1 oflag=direct 
1+0 records in
1+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.0807508 s, 13.0 MB/s
```

#### root

```
arm64@kernel-54:~/hvisor$ dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=1 oflag=direct 
1+0 records in
1+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.0264393 s, 39.7 MB/s
```

简单来看, 速度是qemu的1/4. 

## fio测试工具

fio, Flexible I/O tester, 会创建指定数量的线程进行IO操作, 测试磁盘读写性能. 通过一些参数可以指定fio的测试方法, 这些参数包括:

### 输入参数

* IO类型

**direct**=bool, 是否使用non-buffered IO, 即O_DIRECT

**rw**=str, 指定IO类型, 比如read/write表示顺序读写,randread/readwrite表示随机读写, rw/ randrw表示顺序混合读写和随机混合读写.

* Block大小

**bs**=int, 每个IO操作的块大小

* IO size

**size**=int, 每个线程的file IO的总大小. 单位都是byte

* IO engine

**ioengine**=str, 指定发出IO的操作类型, 例如psync, 则是使用pread和pwrite. libaio则是使用异步IO

* IO depth

**iodepth**=int, If the I/O engine is async, how large a queuing depth do we want to maintain?

* 目标文件

**filename**=str, 指定执行IO操作的对象

* 线程, 进程

**thread**: Fio默认通过fork来创建并执行任务, 如果指定thread, 则会通过线程创建任务

### 输出格式

fio会输出很多内容, 这里逐行分析. 

```c
Jobs: 1 (f=1): [r(1)][100.0%][r=100KiB/s][r=25 IOPS][eta 00m:00s]
```

f=1表示当前打开的文件数, [r(1)]表示当前进行读的任务数量, 100.0%表示任务的完成情况, 之后的两个分别代表带宽和每秒进行的IO总数. eta则是剩余时间.

```c
  read: IOPS=59, BW=240KiB/s (245kB/s)(14.0MiB/60002msec)
  // 读操作, 平均IOPS, 平均带宽()(总的IO大小/总的runtime)
    clat (usec): min=48, max=131459, avg=16623.27, stdev=10650.94
    // Completion latency, 从提交IO给OS到完成IO的时间. 
     lat (usec): min=51, max=131472, avg=16632.89, stdev=10652.58
    // Total latency. Submission latency + Completion latency.
    clat percentiles (usec):
     |  1.00th=[    52],  5.00th=[    61], 10.00th=[   108], 20.00th=[   135],
     | 30.00th=[ 21365], 40.00th=[ 21890], 50.00th=[ 22152], 60.00th=[ 22414],
     | 70.00th=[ 22676], 80.00th=[ 23200], 90.00th=[ 23725], 95.00th=[ 24249],
     | 99.00th=[ 28705], 99.50th=[ 36963], 99.90th=[ 67634], 99.95th=[ 77071],
     | 99.99th=[131597]
   bw (  KiB/s): min=  144, max=  327, per=99.87%, avg=238.70, stdev=31.15, samples=119
   // 记录相等时间间隔下带宽的统计数据, 一般为0.5s. per表示该线程在本组内收到的带宽比例, samples表示样本数
   iops        : min=   36, max=   81, avg=59.22, stdev= 7.74, samples=119
   // 记录相等时间间隔下iops的统计数据
  lat (usec)   : 50=0.45%, 100=6.93%, 250=18.54%, 500=1.28%, 750=0.45%
  lat (usec)   : 1000=0.08%
  lat (msec)   : 50=72.00%, 100=0.25%, 250=0.03%
  cpu          : usr=0.75%, sys=2.23%, ctx=2622, majf=0, minf=1
  // cpu使用率. usr和system time, 该线程经历的上下文切换次数, major page fault的总次数, minor page fault的总次数
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
  // IO深度统计
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
  // How many pieces of I/O were submitting in a single submit call. 这里4表示0-4次就完成了所有submits
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
  // 类似于submit, 这里是complete
     issued rwts: total=3593,0,0,0 short=0,0,0,0 dropped=0,0,0,0
  // The number of read/write/trim requests issued, and how many of them were short or dropped.
     latency   : target=0, window=0, percentile=100.00%, depth=1
 
 // 显示该组的统计信息, bw为总带宽, 最小和最大带宽, io为总的io大小, run表示最小和最大的runtime
 Run status group 0 (all jobs):
   READ: bw=240KiB/s (245kB/s), 240KiB/s-240KiB/s (245kB/s-245kB/s), io=14.0MiB (14.7MB), run=60002-60002msec

// 磁盘统计信息, 用/分割read和write的数据, 分别为:
// ios: 所有groups进行的IO操作次数
// sectors: 表示总IO的扇区数(512B)
// merge: 表示IO调度程序执行的合并数
// ticks表示Number of ticks we kept the disk busy, 
// in_queue: Total time spent in the disk queue, 
// util: The disk utilization. A value of 100% means we kept the disk busy constantly
Disk stats (read/write):
  sda: ios=16398/16511, sectors=32321/65472, merge=30/162, ticks=6853/819634, in_queue=826487, util=100.00%
```

> major page fault是指缺页时页也不在页缓存上, 需要从磁盘调页. minor page fault是指缺页时页在页缓存上, 直接构建地址映射即可. 

### 参考资料

官方文档: https://fio.readthedocs.io/en/latest/fio_doc.html#cmdoption-arg-runtime

CSDN教程: https://blog.csdn.net/weixin_42319496/article/details/119371288

## 测试结果

随机读128MB: 

宿主机: 带宽105MB/s, iops: 25185 (16GB内存, 16核真实CPU) majf=1

root linux: 带宽2300kB/s, iops: 559.81 (768mib内存, 2核CPU) majf=0

non root linux: 带宽245kB/s, iops: 59.22 (768mib内存, 2核CPU)  majf=0

粗略来看, non root是root的十分之一. 

还需要更多测试....

```c
// 命令
fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=128M -numjobs=1 -runtime=80 -group_reporting -name=mytest
// host
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.28
Starting 1 thread
mytest: Laying out IO file (1 file / 128MiB)
Jobs: 1 (f=1)
mytest: (groupid=0, jobs=1): err= 0: pid=187629: Fri Jan 19 10:00:58 2024
  read: IOPS=25.6k, BW=99.8MiB/s (105MB/s)(128MiB/1282msec)
    clat (usec): min=18, max=8688, avg=38.91, stdev=48.07
     lat (usec): min=18, max=8688, avg=38.93, stdev=48.08
    clat percentiles (usec):
     |  1.00th=[   20],  5.00th=[   37], 10.00th=[   37], 20.00th=[   37],
     | 30.00th=[   37], 40.00th=[   37], 50.00th=[   39], 60.00th=[   39],
     | 70.00th=[   39], 80.00th=[   40], 90.00th=[   47], 95.00th=[   48],
     | 99.00th=[   48], 99.50th=[   52], 99.90th=[   65], 99.95th=[   72],
     | 99.99th=[  141]
   bw (  KiB/s): min=95528, max=105952, per=98.53%, avg=100740.00, stdev=7370.88, samples=2
   iops        : min=23882, max=26488, avg=25185.00, stdev=1842.72, samples=2
  lat (usec)   : 20=1.26%, 50=98.17%, 100=0.56%, 250=0.01%, 500=0.01%
  lat (msec)   : 10=0.01%
  cpu          : usr=2.11%, sys=2.97%, ctx=32768, majf=1, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=32768,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=99.8MiB/s (105MB/s), 99.8MiB/s-99.8MiB/s (105MB/s-105MB/s), io=128MiB (134MB), run=1282-1282msec

Disk stats (read/write):
  nvme1n1: ios=32561/25, merge=0/14, ticks=1212/9, in_queue=1223, util=93.16%
// root linux
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1): [r(1)][100.0%][r=2446KiB/s][r=611 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=486: Fri Jan 19 10:07:03 2024
  read: IOPS=561, BW=2246KiB/s (2300kB/s)(128MiB/58367msec)
    clat (usec): min=970, max=31115, avg=1663.72, stdev=796.02
     lat (usec): min=978, max=31123, avg=1675.99, stdev=798.59
    clat percentiles (usec):
     |  1.00th=[ 1156],  5.00th=[ 1221], 10.00th=[ 1254], 20.00th=[ 1303],
     | 30.00th=[ 1369], 40.00th=[ 1418], 50.00th=[ 1434], 60.00th=[ 1500],
     | 70.00th=[ 1598], 80.00th=[ 1778], 90.00th=[ 2409], 95.00th=[ 2671],
     | 99.00th=[ 4948], 99.50th=[ 6718], 99.90th=[10028], 99.95th=[12780],
     | 99.99th=[23462]
   bw (  KiB/s): min=  964, max= 2568, per=99.83%, avg=2241.13, stdev=206.48, samples=116
   iops        : min=  241, max=  642, avg=559.81, stdev=51.60, samples=116
  lat (usec)   : 1000=0.02%
  lat (msec)   : 2=83.55%, 4=15.15%, 10=1.18%, 20=0.09%, 50=0.01%
  cpu          : usr=12.31%, sys=41.42%, ctx=35686, majf=0, minf=2
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=32768,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=2246KiB/s (2300kB/s), 2246KiB/s-2246KiB/s (2300kB/s-2300kB/s), io=128MiB (134MB), run=58367-58367msec

Disk stats (read/write):
  vda: ios=32686/0, merge=0/0, ticks=22952/0, in_queue=64, util=98.23%

// non root
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1): [r(1)][100.0%][r=208KiB/s][r=52 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=171: Thu Jan  1 00:09:08 1970
  read: IOPS=59, BW=240KiB/s (245kB/s)(14.0MiB/60002msec)
    clat (usec): min=48, max=131459, avg=16623.27, stdev=10650.94
     lat (usec): min=51, max=131472, avg=16632.89, stdev=10652.58
    clat percentiles (usec):
     |  1.00th=[    52],  5.00th=[    61], 10.00th=[   108], 20.00th=[   135],
     | 30.00th=[ 21365], 40.00th=[ 21890], 50.00th=[ 22152], 60.00th=[ 22414],
     | 70.00th=[ 22676], 80.00th=[ 23200], 90.00th=[ 23725], 95.00th=[ 24249],
     | 99.00th=[ 28705], 99.50th=[ 36963], 99.90th=[ 67634], 99.95th=[ 77071],
     | 99.99th=[131597]
   bw (  KiB/s): min=  144, max=  327, per=99.87%, avg=238.70, stdev=31.15, samples=119
   iops        : min=   36, max=   81, avg=59.22, stdev= 7.74, samples=119
  lat (usec)   : 50=0.45%, 100=6.93%, 250=18.54%, 500=1.28%, 750=0.45%
  lat (usec)   : 1000=0.08%
  lat (msec)   : 50=72.00%, 100=0.25%, 250=0.03%
  cpu          : usr=0.75%, sys=2.23%, ctx=2622, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=3593,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=240KiB/s (245kB/s), 240KiB/s-240KiB/s (245kB/s-245kB/s), io=14.0MiB (14.7MB), run=60002-60002msec
```

奇怪的现象: 增大IO size, 带宽一直在提高. 打开LOG, 发现一直确实在和root的device通信.

```c
// non root
fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=1G -numjobs=1 -runtime=80 -group_reporting -name=mytest

mytest: (groupid=0, jobs=1): err= 0: pid=164: Thu Jan  1 00:03:42 1970
  read: IOPS=402, BW=1609KiB/s (1648kB/s)(126MiB/80005msec)
    clat (usec): min=46, max=110340, avg=2431.65, stdev=6814.60
     lat (usec): min=49, max=110343, avg=2435.46, stdev=6814.54
    clat percentiles (usec):
     |  1.00th=[   48],  5.00th=[   48], 10.00th=[   49], 20.00th=[   49],
     | 30.00th=[   50], 40.00th=[   50], 50.00th=[   51], 60.00th=[   53],
     | 70.00th=[   58], 80.00th=[  100], 90.00th=[20317], 95.00th=[21103],
     | 99.00th=[22414], 99.50th=[23200], 99.90th=[33162], 99.95th=[39060],
     | 99.99th=[89654]
   bw (  KiB/s): min=  544, max= 2488, per=100.00%, avg=1615.60, stdev=312.61, samples=159
   iops        : min=  136, max=  622, avg=403.76, stdev=78.09, samples=159
  lat (usec)   : 50=39.82%, 100=40.23%, 250=7.58%, 500=1.22%, 750=0.08%
  lat (usec)   : 1000=0.01%
  lat (msec)   : 20=0.23%, 50=10.81%, 100=0.02%, 250=0.01%
  cpu          : usr=1.99%, sys=3.61%, ctx=3619, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=32187,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=1609KiB/s (1648kB/s), 1609KiB/s-1609KiB/s (1648kB/s-1648kB/s), io=126MiB (132MB), run=80005-80005msec

// non root2
fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=4G -numjobs=1 -runtime=20 -group_reporting -name=mytest
       
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
mytest: Laying out IO file (1 file / 4096MiB)
fio: ENOSPC on laying out file, stopping
Jobs: 1 (f=1): [f(1)][100.0%][r=6089KiB/s][r=1522 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=167: Thu Jan  1 00:05:01 1970
  read: IOPS=1427, BW=5709KiB/s (5846kB/s)(112MiB/20003msec)
    clat (usec): min=45, max=38816, avg=642.70, stdev=3507.78
     lat (usec): min=49, max=38820, avg=646.62, stdev=3507.75
    clat percentiles (usec):
     |  1.00th=[   47],  5.00th=[   48], 10.00th=[   48], 20.00th=[   49],
     | 30.00th=[   49], 40.00th=[   49], 50.00th=[   50], 60.00th=[   50],
     | 70.00th=[   51], 80.00th=[   54], 90.00th=[   84], 95.00th=[  122],
     | 99.00th=[21365], 99.50th=[21890], 99.90th=[23200], 99.95th=[30016],
     | 99.99th=[36963]
   bw (  KiB/s): min= 2938, max= 8624, per=100.00%, avg=5839.69, stdev=1220.09, samples=39
   iops        : min=  734, max= 2156, avg=1459.72, stdev=305.03, samples=39
  lat (usec)   : 50=60.28%, 100=33.18%, 250=2.70%, 500=1.05%, 750=0.06%
  lat (usec)   : 1000=0.01%
  lat (msec)   : 20=0.01%, 50=2.72%
  cpu          : usr=6.35%, sys=10.30%, ctx=795, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=28550,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=5709KiB/s (5846kB/s), 5709KiB/s-5709KiB/s (5846kB/s-5846kB/s), io=112MiB (117MB), run=20003-20003msec

// non root 3
fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=10G -numjobs=1 -runtime=20 -group_reporting -name=mytest
       
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
mytest: Laying out IO file (1 file / 10240MiB)
fio: ENOSPC on laying out file, stopping
Jobs: 1 (f=1): [r(1)][100.0%][r=12.6MiB/s][r=3227 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=172: Thu Jan  1 00:08:20 1970
  read: IOPS=3237, BW=12.6MiB/s (13.3MB/s)(253MiB/20002msec)
    clat (usec): min=40, max=73126, avg=270.77, stdev=2147.30
     lat (usec): min=43, max=73129, avg=274.85, stdev=2147.31
    clat percentiles (usec):
     |  1.00th=[   42],  5.00th=[   44], 10.00th=[   45], 20.00th=[   46],
     | 30.00th=[   48], 40.00th=[   48], 50.00th=[   49], 60.00th=[   50],
     | 70.00th=[   50], 80.00th=[   51], 90.00th=[   57], 95.00th=[   87],
     | 99.00th=[18482], 99.50th=[20841], 99.90th=[21890], 99.95th=[22414],
     | 99.99th=[33424]
   bw (  KiB/s): min= 8223, max=18944, per=100.00%, avg=13269.74, stdev=2291.55, samples=39
   iops        : min= 2055, max= 4736, avg=3317.18, stdev=572.92, samples=39
  lat (usec)   : 50=72.92%, 100=23.23%, 250=2.06%, 500=0.72%, 750=0.02%
  lat (usec)   : 1000=0.01%
  lat (msec)   : 20=0.37%, 50=0.67%, 100=0.01%
  cpu          : usr=13.90%, sys=15.53%, ctx=697, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=64766,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=12.6MiB/s (13.3MB/s), 12.6MiB/s-12.6MiB/s (13.3MB/s-13.3MB/s), io=253MiB (265MB), run=20002-20002msec
```

## 宿主机读写文件

1.18

```c
// 命令
fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=128M -numjobs=1 -runtime=80 -group_reporting -name=mytest
// 输出
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.28
Starting 1 thread
mytest: Laying out IO file (1 file / 128MiB)
Jobs: 1 (f=1)
mytest: (groupid=0, jobs=1): err= 0: pid=187629: Fri Jan 19 10:00:58 2024
  read: IOPS=25.6k, BW=99.8MiB/s (105MB/s)(128MiB/1282msec)
    clat (usec): min=18, max=8688, avg=38.91, stdev=48.07
     lat (usec): min=18, max=8688, avg=38.93, stdev=48.08
    clat percentiles (usec):
     |  1.00th=[   20],  5.00th=[   37], 10.00th=[   37], 20.00th=[   37],
     | 30.00th=[   37], 40.00th=[   37], 50.00th=[   39], 60.00th=[   39],
     | 70.00th=[   39], 80.00th=[   40], 90.00th=[   47], 95.00th=[   48],
     | 99.00th=[   48], 99.50th=[   52], 99.90th=[   65], 99.95th=[   72],
     | 99.99th=[  141]
   bw (  KiB/s): min=95528, max=105952, per=98.53%, avg=100740.00, stdev=7370.88, samples=2
   iops        : min=23882, max=26488, avg=25185.00, stdev=1842.72, samples=2
  lat (usec)   : 20=1.26%, 50=98.17%, 100=0.56%, 250=0.01%, 500=0.01%
  lat (msec)   : 10=0.01%
  cpu          : usr=2.11%, sys=2.97%, ctx=32768, majf=1, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=32768,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=99.8MiB/s (105MB/s), 99.8MiB/s-99.8MiB/s (105MB/s-105MB/s), io=128MiB (134MB), run=1282-1282msec

Disk stats (read/write):
  nvme1n1: ios=32561/25, merge=0/14, ticks=1212/9, in_queue=1223, util=93.16%

```



## root的读

```c
// 命令
fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=1M -numjobs=1 -runtime=10 -group_reporting -name=mytest
// 输出: 启动Nonroot前
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1)
mytest: (groupid=0, jobs=1): err= 0: pid=510: Thu Jan 18 20:20:49 2024
  read: IOPS=1108, BW=4433KiB/s (4539kB/s)(1024KiB/231msec)
    clat (usec): min=614, max=8733, avg=770.49, stdev=530.41
     lat (usec): min=619, max=9267, avg=779.11, stdev=561.63
    clat percentiles (usec):
     |  1.00th=[  619],  5.00th=[  635], 10.00th=[  644], 20.00th=[  652],
     | 30.00th=[  660], 40.00th=[  668], 50.00th=[  676], 60.00th=[  725],
     | 70.00th=[  766], 80.00th=[  824], 90.00th=[  881], 95.00th=[  914],
     | 99.00th=[ 1139], 99.50th=[ 3097], 99.90th=[ 8717], 99.95th=[ 8717],
     | 99.99th=[ 8717]
  lat (usec)   : 750=65.62%, 1000=32.03%
  lat (msec)   : 2=1.56%, 4=0.39%, 10=0.39%
  cpu          : usr=14.66%, sys=62.50%, ctx=258, majf=0, minf=2
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=256,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=4433KiB/s (4539kB/s), 4433KiB/s-4433KiB/s (4539kB/s-4539kB/s), io=1024KiB (1049kB), run=231-231msec

Disk stats (read/write):
  vda: ios=251/0, merge=0/0, ticks=73/0, in_queue=0, util=60.47%
// 输出: 启动non root后:
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1)
mytest: (groupid=0, jobs=1): err= 0: pid=539: Thu Jan 18 20:23:26 2024
  read: IOPS=766, BW=3066KiB/s (3139kB/s)(1024KiB/334msec)
    clat (usec): min=795, max=2454, avg=1204.03, stdev=279.86
     lat (usec): min=802, max=2469, avg=1211.91, stdev=281.56
    clat percentiles (usec):
     |  1.00th=[  848],  5.00th=[  938], 10.00th=[  979], 20.00th=[ 1020],
     | 30.00th=[ 1045], 40.00th=[ 1057], 50.00th=[ 1090], 60.00th=[ 1156],
     | 70.00th=[ 1221], 80.00th=[ 1385], 90.00th=[ 1582], 95.00th=[ 1893],
     | 99.00th=[ 2212], 99.50th=[ 2245], 99.90th=[ 2442], 99.95th=[ 2442],
     | 99.99th=[ 2442]
  lat (usec)   : 1000=15.23%
  lat (msec)   : 2=82.03%, 4=2.73%
  cpu          : usr=16.22%, sys=53.75%, ctx=256, majf=0, minf=12
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=256,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=3066KiB/s (3139kB/s), 3066KiB/s-3066KiB/s (3139kB/s-3139kB/s), io=1024KiB (1049kB), run=334-334msec

Disk stats (read/write):
  vda: ios=188/0, merge=0/0, ticks=99/0, in_queue=0, util=64.80%

```

## non root读写文件

non root的内存大小: `0x1fb00000`, file大小1M

```c
# fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=1M -numjobs=1 -runtime=10 -group_reporting -name=mytest
[   39.787015] random: fast init done
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1): [r(1)][100.0%][r=100KiB/s][r=25 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=159: Thu Jan  1 00:00:50 1970
  read: IOPS=23, BW=94.5KiB/s (96.8kB/s)(948KiB/10033msec)
    clat (usec): min=37475, max=91709, avg=42176.53, stdev=5810.85
     lat (usec): min=37480, max=91718, avg=42185.32, stdev=5818.10
    clat percentiles (usec):
     |  1.00th=[38011],  5.00th=[38536], 10.00th=[39060], 20.00th=[39584],
     | 30.00th=[40109], 40.00th=[40109], 50.00th=[40633], 60.00th=[41157],
     | 70.00th=[41681], 80.00th=[42730], 90.00th=[46924], 95.00th=[50070],
     | 99.00th=[69731], 99.50th=[74974], 99.90th=[91751], 99.95th=[91751],
     | 99.99th=[91751]
   bw (  KiB/s): min=   79, max=  103, per=99.15%, avg=93.20, stdev= 6.83, samples=20
   iops        : min=   19, max=   25, avg=22.90, stdev= 1.68, samples=20
  lat (msec)   : 50=94.09%, 100=5.91%
  cpu          : usr=0.27%, sys=1.35%, ctx=242, majf=0, minf=10
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=237,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=94.5KiB/s (96.8kB/s), 94.5KiB/s-94.5KiB/s (96.8kB/s-96.8kB/s), io=948KiB (971kB), run=10033-10033msec
```

non root测试时修改了cell配置文件:

```
non root.cell: 
		/* RAM */ {
			.phys_start = 0x60000000,
			.virt_start = 0x60000000,
			.size = 0x1fb00000,
			.flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
				JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_DMA |
				JAILHOUSE_MEM_LOADABLE,
		},

```

qemu启动命令中命令行参数的内存大小改为512m

***

1.19

内存大小: 768m

```
# fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=128M -numjobs=1 -runtime= -group_reporting -name=mytest

mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1): [r(1)][100.0%][r=172KiB/s][r=43 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=161: Thu Jan  1 00:01:12 1970
  read: IOPS=41, BW=167KiB/s (171kB/s)(1024KiB/6143msec)
    clat (usec): min=21206, max=40145, avg=23838.25, stdev=2288.84
     lat (usec): min=21212, max=40660, avg=23850.66, stdev=2302.93
    clat percentiles (usec):
     |  1.00th=[21365],  5.00th=[21627], 10.00th=[22152], 20.00th=[22676],
     | 30.00th=[23200], 40.00th=[23462], 50.00th=[23725], 60.00th=[23987],
     | 70.00th=[23987], 80.00th=[24249], 90.00th=[25035], 95.00th=[26084],
     | 99.00th=[39060], 99.50th=[39060], 99.90th=[40109], 99.95th=[40109],
     | 99.99th=[40109]
   bw (  KiB/s): min=  158, max=  176, per=99.40%, avg=165.00, stdev= 5.98, samples=12
   iops        : min=   39, max=   44, avg=40.83, stdev= 1.59, samples=12
  lat (msec)   : 50=100.00%
  cpu          : usr=1.27%, sys=2.02%, ctx=261, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=256,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=167KiB/s (171kB/s), 167KiB/s-167KiB/s (171kB/s-171kB/s), io=1024KiB (1049kB), run=6143-6143msec
```

```
# fio -filename=fio_test.img -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=128M -numjobs=1 -runtime=60 -group_reporting -name=mytest
mytest: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.16
Starting 1 thread
Jobs: 1 (f=1): [r(1)][100.0%][r=208KiB/s][r=52 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=171: Thu Jan  1 00:09:08 1970
  read: IOPS=59, BW=240KiB/s (245kB/s)(14.0MiB/60002msec)
    clat (usec): min=48, max=131459, avg=16623.27, stdev=10650.94
     lat (usec): min=51, max=131472, avg=16632.89, stdev=10652.58
    clat percentiles (usec):
     |  1.00th=[    52],  5.00th=[    61], 10.00th=[   108], 20.00th=[   135],
     | 30.00th=[ 21365], 40.00th=[ 21890], 50.00th=[ 22152], 60.00th=[ 22414],
     | 70.00th=[ 22676], 80.00th=[ 23200], 90.00th=[ 23725], 95.00th=[ 24249],
     | 99.00th=[ 28705], 99.50th=[ 36963], 99.90th=[ 67634], 99.95th=[ 77071],
     | 99.99th=[131597]
   bw (  KiB/s): min=  144, max=  327, per=99.87%, avg=238.70, stdev=31.15, samples=119
   iops        : min=   36, max=   81, avg=59.22, stdev= 7.74, samples=119
  lat (usec)   : 50=0.45%, 100=6.93%, 250=18.54%, 500=1.28%, 750=0.45%
  lat (usec)   : 1000=0.08%
  lat (msec)   : 50=72.00%, 100=0.25%, 250=0.03%
  cpu          : usr=0.75%, sys=2.23%, ctx=2622, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=3593,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=240KiB/s (245kB/s), 240KiB/s-240KiB/s (245kB/s-245kB/s), io=14.0MiB (14.7MB), run=60002-60002msec

```

```
fio -filename=/dev/vda -direct=1 -iodepth 1 -thread -rw=rw -ioengine=psync -bs=4k -size=128M -numjobs=1 -runtime=80 -group_reporting -name=mytest
```

