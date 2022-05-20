---
title: "load average 过高, mount nfs 问题处理"
description: "load average 过高 mount nfs 问题处理"
date: "2022-03-14"
menu: "main"
tags:
- "nfs"
- "kubernetes"
categories:
- "technology"
---

周末, 有一台服务器告警: 系统负载过高, 最高的时候都已经到 100 +, 以下是排查&处理的具体过程.

## 发现的问题/现象

### `uptime` 显示 load average 都在70+

因为服务器是40核心, 原则上负载40是满负荷, 现在明显存在大量等待的任务. 继续往下分析进程, 看具体那个进程一直在堵塞.

### `ps -ef` 执行到某一个进程就卡住了

命令执行如下:

```sh
$ ps -ef 
...
root  40004  2912  0  Mar08  ?  00:00:33  containerd-shim -namespace moby -workdir /data/docker/containerd/daemon/  
io.containerd.runtime.v1.linux/moby/<container-hash>
卡住了
```

根据命令中的 <container-hash> 找到对应的 pod, 将其从当前节点移除. 移除之后, ps 命令以及其他系统命令可以成功执行. 被移除的 pod 分别是: 2个 prometheus、 1个 mysql.

### 无法执行 umount 卸载

测试 mount 挂载正常, 但是 umount 失败, 解决办法:

- 先强制 umount

```sh
$ umount  -f -l /mount-point

# 命令解释
$ umount [options] <source> | <directory>

Options:
-f   Force unmount (in case of an unreachable NFS system). (Requires kernel 
2.1.116 or later.)
-l   Lazy unmount. Detach the filesystem from the filesystem hierarchy now, 
and cleanup all references to the filesystem as soon as it is not busy 
anymore. (Requires kernel 2.4.11 or later.)
```
> https://linux.die.net/man/8/umount

- kill占用进程

```sh
$ fuser –m –v /mount-point
USER        PID  ACCESS COMMAND
/mount-point:
...
user1      21691 .rce.  ls
...

# 必须 -9
$ kill -9 21691
```

## 继续分析

通过阅读其他文档发现:

### 网络原因导致某连接断开, 该连接进入持续的等待中

这种情况会提高负载, 这里涉及一个 nfs 挂载参数, 通过 `mount | grep nfs` 看到 kubelet 默认的挂载配置如下:

```sh
$ mount | grep nfs
10.***.***.6:/3PAR_d11_Node1_FPG/3PAR_d11_Node1_VFS/paas_share_FS/***/fz***c on 
/var/lib/kubelet/pods/c6c26172-1030-4dad-8611-102636803a58/volumes/kubernetes.io
~nfs/pvc-a7e8bf54-c787-44a4-94d5-b849ec2d24bb 
type nfs4 
(rw,relatime,vers=4.0,rsize=1048576,wsize=1048576,
namlen=255,hard,proto=tcp,timeo=600,retrans=2,
sec=sys,clientaddr=10.***.***.140,local_lock=none,
addr=10.***.***.6)
...
```

括号中可以看到: rw, nfs4.0, hard, tcp 等配置信息, 简单说一下 mount 配置:

```sh
mount -F nfs [-o mount-options] server:/directory /mount-point
# 比如:
mount -F nfs -o hard 192.168.0.10:/nfs /nfs
# 同时使用多个参数使用逗号分隔：
mount -t nfs -o timeo=3,udp,hard 192.168.0.30:/tmp /nfs
```

`-o mount-options` 指定可以用来挂载 NFS 文件系统的挂载选项。有关常用的 mount 选项的列表，请参见[表 19–2](https://docs.oracle.com/cd/E19253-01/819-7062/6n91k1fr7/index.html#fsmount-66498)  
`-o hard/soft` 如果服务器没有响应，有 hard 和 soft 两种处理方式, soft 选项表示返回了错误, hard 选项表示继续重试请求, 直到服务器响应为止. 缺省情况下使用 hard.

详细解释如下:

```
soft / hard

Determines the recovery behavior of the NFS client after an NFS request times 
out. If neither option is specified (or if the hard option is specified), 
NFS requests are retried indefinitely. If the soft option is specified, 
then the NFS client fails an NFS request after retrans retransmissions 
have been sent, causing the NFS client to return an error to the calling 
application.  

NB: A so-called "soft" timeout can cause silent data corruption in certain 
cases. As such, use the soft option only when client responsiveness is more 
important than data integrity. Using NFS over TCP or increasing the value of 
the retrans option may mitigate some of the risks of using the soft option.  
```
> https://linux.die.net/man/5/nfs

在某些情况下, soft 模式可能会导致静默数据损坏. 因此，仅当客户端响应比数据完整性更重要时才使用 soft. 考虑到 nfs 作为 kubernetes 持久卷使用的话, 数据完整性肯定是比较重要的, 所以还是沿用官方默认配置 hard. 并未做修改.

### 官方不建议使用 nfs 作为 prometheus 后端存储

阅读prometheus 文档发现:

```
CAUTION:
Non-POSIX compliant filesystems are not supported for Prometheus' local 
storage as unrecoverable corruptions may happen. NFS filesystems (including 
AWS's EFS) are not supported. NFS could be POSIX-compliant, but most 
implementations are not. It is strongly recommended to use a local filesystem 
for reliability.
```
> https://prometheus.io/docs/prometheus/latest/storage/

听人劝吃饱饭, 文档中建议用本地文件系统, 所以使用 [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner) 替换了 prometheus 原来的 nfs 后端存储.