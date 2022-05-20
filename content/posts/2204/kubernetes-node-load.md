---
title: "分析告警 kubernetes 节点 load 过高问题"
description: "分析 kubernetes 节点 load 过高的问题，并自研程序找出问题 pod 发出告警"
summary: ""
date: "2022-04-27"
menu: "main"
bookToC: false
tags:
- kubernetes
categories:
- "technology"
---

## 负载过高分析

通过 linux 提供的几个命令可以从不同的纬度分析系统负载。

### vmstat

这命令能从一个系统的角度反应出服务器情况，报告虚拟内存统计信息，报告有关进程、内存、分页、块的信息 IO、陷阱、磁盘和 CPU 活动。看个例子：

```shell
$ vmstat --wide --unit M 5
procs ----------------memory---------------- ---swap--- -----io---- ---system--- ---------cpu--------
 r  b     swpd      free      buff     cache   si   so    bi    bo     in   cs     us sy  id  wa  st
 1  1        0    127691      1535     73572    0    0     0     3      0    0     2   1  97   0   0
93  0        0    127674      1535     73573    0    0     0    80   49267 67634   5   1  94   1   0
 0  2        0    127679      1535     73573    0    0     0    66   38537 56283   3   1  95   1   0
 2  2        0    127738      1535     73574    0    0     6    86   41769 61823   5   1  93   2   0
 2  0        0    127729      1535     73574    0    0    18    18   41002 59214   4   1  95   0   0
```

命令以及输出解释：

```
vmstat
- --wide 宽幅展示 比较易读  
- --unit 输出单位，可以是 1000(k)、1024(K)、1000000(m) 或 1048576(M) 个字节

procs
- r: 可运行 runnable 进程数量（包括运行中 running 或者等待中 waiting 的进程）  
- b: 等待 i/o 完成的阻塞 blocked 进程数

memory
显示单位受 --unit 影响
- swpd: 交换 swap 内存使用量
- free: 空闲 idle 内存数
- buff: 用作缓冲区 buffers 的内存量
- cache: 用作缓存 cache 的内存量

swap
显示单位受 --unit 影响
- si: 每秒从磁盘换入 swapped 的内存量
- so: 每秒交换 swapped 到磁盘的内存量

io
- bi: 从块设备接收的块 block (blocks/s)
- bo: 发送到块设备到块 block (blocks/s)

system
- in：每秒的中断数，包括时钟 clock
- cs：每秒上下文 context 切换的次数。

cpu
以下都是 cpu 总时间的百分比
- us: 非内核代码运行耗时 (user time & nice time)
- sy: 内核代码运行耗时 (系统耗时)
- id: 空闲 idle 时间，从 Linux 2.5.41开始它包含 i/o 等待时间
- *wa: io 等待时间，从 Linux 2.5.41开始它包含在 idle 中
- *st: 从虚拟机中窃取时间，未知
```

这个例子中，可见 cpu 空闲 idle 时间占比 90% 以上，说明 i/o 等待时间很高。

### iostat

cpu 统计报告 & 设备/分区 input/output 统计报告。

```shell
$ iostat -m 2 3
Linux 5.4.108-1.el7.elrepo.x86_64 (node27)  04/28/2022   _x86_64_   (80 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.35    0.05    0.64    0.21    0.00   96.75

Device:            tps    MB_read/s    MB_wrtn/s    MB_read    MB_wrtn
sdb               5.13         0.00         0.19       3477    5627354
sda               3.10         0.00         0.03       1890     738773

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.78    0.00    0.57    1.30    0.00   96.34

Device:            tps    MB_read/s    MB_wrtn/s    MB_read    MB_wrtn
sdb               0.00         0.00         0.00          0          0
sda               0.50         0.00         0.00          0          0

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.71    0.01    0.61    0.94    0.00   96.72

Device:            tps    MB_read/s    MB_wrtn/s    MB_read    MB_wrtn
sdb               0.00         0.00         0.00          0          0
sda               0.00         0.00         0.00          0          0
```

报告解读：

```
iostat -m 2 3
- -m: 以兆为单位显示
- -x: 展示扩展信息 
- 每隔2秒输出一次报告，总共输出3次。
- 第一份报告是系统自启动以来的统计信息，每个后续报告都是上次报告以来的统计信息。

avg-cpu
cpu 使用率报告，对于多核心系统，这里的值是全局平均值。下面每一项都是对比 cpu 总时间的使用率。
- %user: 执行用户级别应用所用的时间
- %nice: 执行具有 nice 优先级的用户级别程序所用的时间
- %system: 执行系统级别（内核）程序所用时间
- %iowait:  cpu/cpus 空闲等待磁盘 i/o 请求所用的时间
- %steal: 当虚拟机管理器正服务另一个虚拟处理器时，虚拟 cpu/cpus 的被迫等待时间（被偷走的时间）
- %idle: 系统没有 i/o 请求时的 cpu/cpus 空闲时间

下面是设备使用率报告
展示每一个物理设备/分区的统计信息
- tps: 每秒向设备发出的传输次数。一次传输就是一个 i/o 请求，多个逻辑请求可以合并成一次 i/o 请求。
- MB_read/s, MB_wrtn/s: 每秒读取/写入设备的数据量，单位 M
- MB_read, MB_wrtn: 读取/写入设备的总数据量
```

从例子中，可见 cpu %idle 占比 90% 以上，说明 cpu 花了绝大多数时间在等待 i/o。设备的读写数据几乎没有，说明这些 i/o 并不是来自系统物理设备/分区。有可能来自挂载的网络文件存储设备。

### ps

当前进程信息快照，通过下面的命令找出存在大量 io 的进程

```shell
$ ps -e -L o state,pid,cmd | grep "^[R|D]" | sort | uniq -c | sort -k 1nr
41 R 75319 /bin/node_exporter...
```

命令解释：

```shell
-e every 输出所有进程
o 自定义输出列 逗号分割
-L 展示线程

进程状态码介绍
- R 在执行队列中的，正在运行中或者可以运行的 running or runnable
- D 不可中断的睡眠 (通常是 i/o)
- S 可以中断的睡眠 (正在等待某个事件完成)
- I 空闲 Idle 的内核线程
- s session leader
- < 高优先级
- + 在前台进程组中
- l 是多线程的
```

使用下面的命令找出占用 io 的 pod uid

```shell
$ cat /proc/75319/cgroup | awk -F "/" '{print $4}'
pode0c67fad-9fab-4f35-87b3-d918b5f09882
...

$ kubectl get pods --all-namespaces \
  -ocustom-columns=NS:metadata.namespace,Name:metadata.name,UID:metadata.uid \
  | grep e0c67fad-9fab-4f35-87b3-d918b5f09882
```

