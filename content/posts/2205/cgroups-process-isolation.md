---
title: "Linux 控制组(cgroups)和进程隔离"
description: "本文详细介绍了cgroups的概念和技术原理"
summary: "控制组(cgroups)是内核的一个特性，它能限制/统计/隔离一个或者多个进程使用CPU、内存、磁盘I/O和网络。cgroups技术最开始是Google开发，最终在2.6.24版本的内核中出现。3.15和3.16版本内核将合并进重新设计的cgroups，它添加来kernfs(拆分一些sysfs逻辑)。cgroups的主要设计目标是提供一个统一的接口，它可以管理进程或者整个操作系统级别的虚拟化，包含Linux容器，或者LXC。"
date: "2022-05-15"
menu: "main"
tags:
- cgroups
- kubernetes
categories:
- "technology"
---

每个人都听过容器，但它究竟是什么？

支持这项技术的软件有很多，其中 Docker 最为流行。因为它的可移植性和环境隔离的能力，它在数据中心内部特别流行。为了能理解这个技术，需要理解很多方面。

注意：很多人拿容器和虚拟机比较，他们有不同的设计目标，不是替代关系，重叠度很小。容器旨在成为一个轻量级环境，您可以裸机上启动容器，托管一个或几个独立的应用程序。当您想要托管整个操作系统或生态系统或者可能运行与底层环境不兼容的应用程序时，您应该选择虚拟机。

## Linux 控制组

说实话，零信任环境下有些软件的确需要被控制或被限制 - 至少为了稳定，或是为了安全。很多时候一个Bug或不良代码可能会摧毁整个机器并削弱整个生态系统。还好，有办法来控制这些应用程序，控制组(cgroups)是内核的一个特性，它能限制/计量/隔离一个或者多个进程使用CPU、内存、磁盘I/O和网络。  
cgroup技术最开始是Google开发，最终在2.6.24版本（2008年1月）的内核中出现。3.15和3.16版本内核将合并进重新设计的cgroups，它添加了kernfs(拆分一些sysfs逻辑)。  
cgroups的主要设计目标是提供一个统一的接口，它可以管理进程或者整个操作系统级别的虚拟化，包含Linux容器，或者LXC。cgroups主要提供了以下能力：

- **资源限制**：一个组，可以通过配置使其不能使用超过特定内存限制，或者使用超过指定数量的处理器，或者被限制使用特定的外围设备。
- **优先级**：可以配置一个或多个组比别的组使用更少/更多的CPU或者I/O吞吐。
- **计量**：组的资源使用是被监控和计量的。
- **控制**：进程组可以被冻结、停止或重启。

一个 cgroup 可以由一个或多个进程组成，这些进程都绑定到同一组限制。这些组也可以是分层的，这意味着子组继承了对其父组管理的限制。  
Linux内核为cgroups提供了一系列控制器或者子系统，控制器负责给一个或者一组进程分配指定的系统资源。比如，`memory`控制器限制内存使用，`cpuacct`控制器限制cpu使用。  
您可以直接或间接访问和管理 cgroup（使用 LXC、libvirt 或 Docker），首先，我在这里通过 sysfs 和 `libcgroups` 库介绍。下面的例子中，需要安装必要的软件包。在Red Hat Enterprise Linux或者CentOS上，执行下面命令：

```shell
sudo yum install libcgroup libcgroup-tools
```

在Ubuntu或Debian上这样安装：

```shell
sudo apt-get install libcgroup1 cgroup-tools
```

这个例子中，我用一个简单的脚本(test.sh)，里面会执行一个无限循环。

```shell
$ cat test.sh
#!/bin/sh

while [ 1 ]; do
    echo "hello world"
    sleep 60
done
```

## 手动方式

需要的软件包安装完毕之后，您可以通过 **sysfs 层次结构**直接配置您的 cgroup。比如，要在`memory`子系统下创建一个名为 `foo` 的 cgroup，请在 `/sys/fs/cgroup/memory` 中创建一个名为 foo 的目录：

```shell
sudo mkdir /sys/fs/cgroup/memory/foo
```

默认情况下，每个新创建的 cgroup 都将继承对系统整个内存池的访问权限。但是，对于那些不断分配内存却不释放的应用来说，这样并不好。要将应用程序限制在合理的范围内，您需要更新 memory.limit_in_bytes 文件。

```shell
$ echo 50000000 | sudo tee
 ↪/sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```

验证配置：

```shell
$ sudo cat memory.limit_in_bytes
50003968
```

注意，读到的值通常是内核页大小的倍数(page size, 4096bytes 或 4KB)。  
执行应用程序：

```shell
$ sh ~/test.sh &
```

使用该进程PID，将其添加到`memory`控制器管理下，

```shell
$ echo 2845 > /sys/fs/cgroup/memory/foo/cgroup.procs
```

使用相同的 PID 号，列出正在运行的进程，并验证它是否在期望的 cgroup 中运行：

```shell
$ ps -o cgroup 2845
CGROUP
8:memory:/foo,1:name=systemd:/user.slice/user-0.slice/
↪session-4.scope
```

您还可以通过读取指定的文件来监控该 cgroup 当前使用的资源。在这个例子中，你可能想看一下当前进程(以及派生的子进程)的内存使用量。

```shell
$ cat /sys/fs/cgroup/memory/foo/memory.usage_in_bytes
253952
```

## 当程序不良运行

还是上面的例子，我们将`cgroup/foo`内存限制调整为 500 bytes。

```shell
$ echo 500 | sudo tee /sys/fs/cgroup/memory/foo/
↪memory.limit_in_bytes
```
*注意：如果一个任务超出了其定义的限制，内核将进行干预，在某些情况下，会终止该任务。*  
同样，再读这个值，因为它要是内核页大小的倍数。所以尽管你配置的是500字节，但实际上设置的是4KB。

```shell
$ cat /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
4096
```

启动应用，将其移动到cgroup中，并监控系统日志。

```shell
$ sudo tail -f /var/log/messages

Oct 14 10:22:40 localhost kernel: sh invoked oom-killer:
 ↪gfp_mask=0xd0, order=0, oom_score_adj=0
Oct 14 10:22:40 localhost kernel: sh cpuset=/ mems_allowed=0
Oct 14 10:22:40 localhost kernel: CPU: 0 PID: 2687 Comm:
 ↪sh Tainted: G
OE  ------------   3.10.0-327.36.3.el7.x86_64 #1
Oct 14 10:22:40 localhost kernel: Hardware name: innotek GmbH
VirtualBox/VirtualBox, BIOS VirtualBox 12/01/2006
Oct 14 10:22:40 localhost kernel: ffff880036ea5c00
 ↪0000000093314010 ffff88000002bcd0 ffffffff81636431
Oct 14 10:22:40 localhost kernel: ffff88000002bd60
 ↪ffffffff816313cc 01018800000000d0 ffff88000002bd68
Oct 14 10:22:40 localhost kernel: ffffffffbc35e040
 ↪fffeefff00000000 0000000000000001 ffff880036ea6103
Oct 14 10:22:40 localhost kernel: Call Trace:
Oct 14 10:22:40 localhost kernel: [<ffffffff81636431>]
 ↪dump_stack+0x19/0x1b
Oct 14 10:22:40 localhost kernel: [<ffffffff816313cc>]
 ↪dump_header+0x8e/0x214
Oct 14 10:22:40 localhost kernel: [<ffffffff8116d21e>]
 ↪oom_kill_process+0x24e/0x3b0
Oct 14 10:22:40 localhost kernel: [<ffffffff81088e4e>] ?
 ↪has_capability_noaudit+0x1e/0x30
Oct 14 10:22:40 localhost kernel: [<ffffffff811d4285>]
 ↪mem_cgroup_oom_synchronize+0x575/0x5a0
Oct 14 10:22:40 localhost kernel: [<ffffffff811d3650>] ?
 ↪mem_cgroup_charge_common+0xc0/0xc0
Oct 14 10:22:40 localhost kernel: [<ffffffff8116da94>]
 ↪pagefault_out_of_memory+0x14/0x90
Oct 14 10:22:40 localhost kernel: [<ffffffff8162f815>]
 ↪mm_fault_error+0x68/0x12b
Oct 14 10:22:40 localhost kernel: [<ffffffff816422d2>]
 ↪__do_page_fault+0x3e2/0x450
Oct 14 10:22:40 localhost kernel: [<ffffffff81642363>]
 ↪do_page_fault+0x23/0x80
Oct 14 10:22:40 localhost kernel: [<ffffffff8163e648>]
 ↪page_fault+0x28/0x30
Oct 14 10:22:40 localhost kernel: Task in /foo killed as
 ↪a result of limit of /foo
Oct 14 10:22:40 localhost kernel: memory: usage 4kB, limit
 ↪4kB, failcnt 8
Oct 14 10:22:40 localhost kernel: memory+swap: usage 4kB,
 ↪limit 9007199254740991kB, failcnt 0
Oct 14 10:22:40 localhost kernel: kmem: usage 0kB, limit
 ↪9007199254740991kB, failcnt 0
Oct 14 10:22:40 localhost kernel: Memory cgroup stats for /foo:
 ↪cache:0KB rss:4KB rss_huge:0KB mapped_file:0KB swap:0KB
 ↪inactive_anon:0KB active_anon:0KB inactive_file:0KB
 ↪active_file:0KB unevictable:0KB
Oct 14 10:22:40 localhost kernel: [ pid ]   uid  tgid total_vm
 ↪rss nr_ptes swapents oom_score_adj name
Oct 14 10:22:40 localhost kernel: [ 2687]     0  2687    28281
 ↪347     12        0             0 sh
Oct 14 10:22:40 localhost kernel: [ 2702]     0  2702    28281
 ↪50    7        0             0 sh
Oct 14 10:22:40 localhost kernel: Memory cgroup out of memory:
 ↪Kill process 2687 (sh) score 0 or sacrifice child
Oct 14 10:22:40 localhost kernel: Killed process 2702 (sh)
 ↪total-vm:113124kB, anon-rss:200kB, file-rss:0kB
Oct 14 10:22:41 localhost kernel: sh invoked oom-killer:
 ↪gfp_mask=0xd0, order=0, oom_score_adj=0
[ ... ]
```

注意，一旦应用程序使用内存达到 4KB 限制，内核的 Out-Of-Memory Killer（或 oom-killer）就会介入。它杀死了应用程序。您可以下面的方式来验证这一点：

```shell
$ ps -o cgroup 2687
CGROUP
```

## 使用 libcgroup

`libcgroup`软件包提供了简单的管理工具，上面很多操作步骤都可以用它实现。例如，使用cgcreate命令可以创建sysfs条目和文件。  
在`memory`子系统下创建名字为foo的组，使用下面命令：  
```shell
$ sudo cgcreate -g memory:foo
```
*注意：libcgroup 提供了一种用于管理**控制组**中的任务的机制。*  
使用与之前相同的方法，设置阈值：  
```shell
$ echo 50000000 | sudo tee
 ↪/sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```
验证配置：  
```shell
$ sudo cat memory.limit_in_bytes
50003968
```
使用 cgexec 命令在 `cgroup/foo` 下运行应用程序：  
```shell
$ sudo cgexec -g memory:foo ~/test.sh
```
使用它的 PID，验证应用程序是否在 cgroup 和定义的`memory`管理器下运行：  
```shell
$  ps -o cgroup 2945
CGROUP
6:memory:/foo,1:name=systemd:/user.slice/user-0.slice/
↪session-1.scope
```
如果您的应用程序不再运行，并且您想要清理并删除 cgroup，您可以使用 cgdelete。要从`memory`控制器下删除组 foo，请键入：
```shell
$ sudo cgdelete memory:foo
```

## 持久组

通过简单的配置文件来启动服务，也可以完成上面的工作。你可以在`/etc/cgconfig.conf`文件中定义所有cgroup名字和属性。下面的例子中配置了foo组和它的一些属性。  
```shell
$ cat /etc/cgconfig.conf
#
#  Copyright IBM Corporation. 2007
#
#  Authors:     Balbir Singh <balbir@linux.vnet.ibm.com>
#  This program is free software; you can redistribute it
#  and/or modify it under the terms of version 2.1 of the GNU
#  Lesser General Public License as published by the Free
#  Software Foundation.
#
#  This program is distributed in the hope that it would be
#  useful, but WITHOUT ANY WARRANTY; without even the implied
#  warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR
#  PURPOSE.
#
# 默认，我们希望 systemd 默认加载所有内容
# 所以没啥可做的
# 详细内容查看 man cgconfig.conf
# 了解如何在系统启动时使用该文件创建 cgroup

group foo {
  cpu {
    cpu.shares = 100;
  }
  memory {
    memory.limit_in_bytes = 5000000;
  }
}
```

`cpu.shares`定义了cgroup的CPU优先级。默认，所有的组继承 1024 shares 或者说 100% CPU使用时间。降低该值，比如 100，该组将被限制在大约 10% CPU使用时间。  
如前所述，cgroup 中的进程也可以被限制使用CPUs(core)数量，把下面的内容添加到cgconfig.conf文件相应的cgroup下：

```shell
cpuset {
  cpuset.cpus="0-5";
}
```
它将限制该cgroup使用索引为0到5的核心(core)，即仅能使用前6个CPU核心。  
下面，需要使用`cgconfig`服务加载该配置文件。首先，配置`cgconfig`开机自启动加载上面的配置文件。

```shell
$ sudo systemctl enable cgconfig
Create symlink from /etc/systemd/system/sysinit.target.wants/
↪cgconfig.service
to /usr/lib/systemd/system/cgconfig.service.
```
现在，手动启动服务加载配置文件（或者直接重启操作系统）  
```shell
$ sudo systemctl start cgconfig
```
在`cgroup/foo`下，启动应用，并将其和它的`memory`、`cpuset`和`cpu`限制进行绑定：  
```shell
$ sudo cgexec -g memory,cpu,cpuset:foo ~/test.sh &
```
除了将应用启动到预定义的cgroup中之外，剩下的操作系统重启后会一直存在。但是，你可以通过写一个依赖`cgconfig`服务的开机初始化脚本来启动应用。