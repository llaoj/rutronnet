---
title: "bcc 之 opensnoop 工具的使用"
description: "bcc 之 opensnoop 工具的使用文档"
summary: "这篇文档主要演示了 opensnoop(Linux eBPF/bcc) 工具的使用. opensnoop 在系统范围内跟踪 open() 系统调用，并打印各种详细信息."
date: "2022-03-16"
menu: "main"
tags:
- "bcc"
- "ebpf"
categories:
- "technology"
---

这篇文档主要演示了 opensnoop(Linux eBPF/bcc) 工具的使用.

## 示例

opensnoop 在系统范围内跟踪 open() 系统调用，并打印各种详细信息.

示例输出:

```
# ./opensnoop
PID    COMM      FD ERR PATH
17326  <...>      7   0 /sys/kernel/debug/tracing/trace_pipe
1576   snmpd      9   0 /proc/net/dev
1576   snmpd     11   0 /proc/net/if_inet6
1576   snmpd     11   0 /proc/sys/net/ipv4/neigh/eth0/retrans_time_ms
1576   snmpd     11   0 /proc/sys/net/ipv6/neigh/eth0/retrans_time_ms
1576   snmpd     11   0 /proc/sys/net/ipv6/conf/eth0/forwarding
1576   snmpd     11   0 /proc/sys/net/ipv6/neigh/eth0/base_reachable_time_ms
1576   snmpd     11   0 /proc/sys/net/ipv4/neigh/lo/retrans_time_ms
1576   snmpd     11   0 /proc/sys/net/ipv6/neigh/lo/retrans_time_ms
1576   snmpd     11   0 /proc/sys/net/ipv6/conf/lo/forwarding
1576   snmpd     11   0 /proc/sys/net/ipv6/neigh/lo/base_reachable_time_ms
1576   snmpd      9   0 /proc/diskstats
1576   snmpd      9   0 /proc/stat
1576   snmpd      9   0 /proc/vmstat
1956   supervise  9   0 supervise/status.new
1956   supervise  9   0 supervise/status.new
17358  run        3   0 /etc/ld.so.cache
17358  run        3   0 /lib/x86_64-linux-gnu/libtinfo.so.5
17358  run        3   0 /lib/x86_64-linux-gnu/libdl.so.2
17358  run        3   0 /lib/x86_64-linux-gnu/libc.so.6
17358  run       -1   6 /dev/tty
17358  run        3   0 /proc/meminfo
17358  run        3   0 /etc/nsswitch.conf
17358  run        3   0 /etc/ld.so.cache
17358  run        3   0 /lib/x86_64-linux-gnu/libnss_compat.so.2
17358  run        3   0 /lib/x86_64-linux-gnu/libnsl.so.1
17358  run        3   0 /etc/ld.so.cache
17358  run        3   0 /lib/x86_64-linux-gnu/libnss_nis.so.2
17358  run        3   0 /lib/x86_64-linux-gnu/libnss_files.so.2
17358  run        3   0 /etc/passwd
17358  run        3   0 ./run
^C
```

在跟踪时，snmpd 进程打开了各种 /proc 文件(读取指标). 另外, 一个 “run” 进程读取各种库和配置文件(看起来像
正在启动: 一个新进程).

如果在应用程序启动期间使用, **opensnoop 可用于发现配置和日志文件**.


## 过滤 PID

-p 选项可用于在内核中过滤 PID. 这里我将它与 -T 一起使用来打印时间戳:

```
$ ./opensnoop -Tp 1956
TIME(s)       PID    COMM               FD ERR PATH
0.000000000   1956   supervise           9   0 supervise/status.new
0.000289999   1956   supervise           9   0 supervise/status.new
1.023068000   1956   supervise           9   0 supervise/status.new
1.023381997   1956   supervise           9   0 supervise/status.new
2.046030000   1956   supervise           9   0 supervise/status.new
2.046363000   1956   supervise           9   0 supervise/status.new
3.068203997   1956   supervise           9   0 supervise/status.new
3.068544999   1956   supervise           9   0 supervise/status.new
```

这表明 supervise 进程每秒打开2次 status.new 文件.


## 包含/过滤 UID

-U 选项在输出中包含 UID:

```
# ./opensnoop -U
UID   PID    COMM               FD ERR PATH
0     27063  vminfo              5   0 /var/run/utmp
103   628    dbus-daemon        -1   2 /usr/local/share/dbus-1/system-services
103   628    dbus-daemon        18   0 /usr/share/dbus-1/system-services
103   628    dbus-daemon        -1   2 /lib/dbus-1/system-services
```

-u 选项过滤 UID:

```
# ./opensnoop -Uu 1000
UID   PID    COMM               FD ERR PATH
1000  30240  ls                  3   0 /etc/ld.so.cache
1000  30240  ls                  3   0 /lib/x86_64-linux-gnu/libselinux.so.1
1000  30240  ls                  3   0 /lib/x86_64-linux-gnu/libc.so.6
1000  30240  ls                  3   0 /lib/x86_64-linux-gnu/libpcre.so.3
1000  30240  ls                  3   0 /lib/x86_64-linux-gnu/libdl.so.2
1000  30240  ls                  3   0 /lib/x86_64-linux-gnu/libpthread.so.0
```


## 过滤失败的 opens

-x 选项仅打印失败的 opens:

```
# ./opensnoop -x
PID    COMM      FD ERR PATH
18372  run       -1   6 /dev/tty
18373  run       -1   6 /dev/tty
18373  multilog  -1  13 lock
18372  multilog  -1  13 lock
18384  df        -1   2 /usr/share/locale/en_US.UTF-8/LC_MESSAGES/coreutils.mo
18384  df        -1   2 /usr/share/locale/en_US.utf8/LC_MESSAGES/coreutils.mo
18384  df        -1   2 /usr/share/locale/en_US/LC_MESSAGES/coreutils.mo
18384  df        -1   2 /usr/share/locale/en.UTF-8/LC_MESSAGES/coreutils.mo
18384  df        -1   2 /usr/share/locale/en.utf8/LC_MESSAGES/coreutils.mo
18384  df        -1   2 /usr/share/locale/en/LC_MESSAGES/coreutils.mo
18385  run       -1   6 /dev/tty
18386  run       -1   6 /dev/tty
```

这里捕获了一个 df 命令无法打开 coreutils.mo 文件, 并尝试从其他目录打开.

列 ERR 表示系统错误码, 2 代表 ENOENT: no such file or directory.


可以使用 -d 选项设置最长跟踪持续时间. 例如, 要跟踪 2秒:

```
# ./opensnoop -d 2
PID    COMM               FD ERR PATH
2191   indicator-multi    11   0 /sys/block
2191   indicator-multi    11   0 /sys/block
2191   indicator-multi    11   0 /sys/block
2191   indicator-multi    11   0 /sys/block
2191   indicator-multi    11   0 /sys/block
```


## 过滤进程名称

-n 选项可用于过滤进程名称(部分匹配):

```
# ./opensnoop -n ed

PID    COMM               FD ERR PATH
2679   sed                 3   0 /etc/ld.so.cache
2679   sed                 3   0 /lib/x86_64-linux-gnu/libselinux.so.1
2679   sed                 3   0 /lib/x86_64-linux-gnu/libc.so.6
2679   sed                 3   0 /lib/x86_64-linux-gnu/libpcre.so.3
2679   sed                 3   0 /lib/x86_64-linux-gnu/libdl.so.2
2679   sed                 3   0 /lib/x86_64-linux-gnu/libpthread.so.0
2679   sed                 3   0 /proc/filesystems
2679   sed                 3   0 /usr/lib/locale/locale-archive
2679   sed                -1   2
2679   sed                 3   0 /usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache
2679   sed                 3   0 /dev/null
2680   sed                 3   0 /etc/ld.so.cache
2680   sed                 3   0 /lib/x86_64-linux-gnu/libselinux.so.1
2680   sed                 3   0 /lib/x86_64-linux-gnu/libc.so.6
2680   sed                 3   0 /lib/x86_64-linux-gnu/libpcre.so.3
2680   sed                 3   0 /lib/x86_64-linux-gnu/libdl.so.2
2680   sed                 3   0 /lib/x86_64-linux-gnu/libpthread.so.0
2680   sed                 3   0 /proc/filesystems
2680   sed                 3   0 /usr/lib/locale/locale-archive
2680   sed                -1   2
^C
```

这里捕获了 “sed” 命令，是因为命令中使用了命令名称部分匹配 “-n ed”


## 过滤标志

-e 选项能打印出额外的列; 例如，以下输出包含传递给 open(2) 的标志(以八进制表示):

```
# ./opensnoop -e
PID    COMM               FD ERR FLAGS    PATH
28512  sshd               10   0 00101101 /proc/self/oom_score_adj
28512  sshd                3   0 02100000 /etc/ld.so.cache
28512  sshd                3   0 02100000 /lib/x86_64-linux-gnu/libwrap.so.0
28512  sshd                3   0 02100000 /lib/x86_64-linux-gnu/libaudit.so.1
28512  sshd                3   0 02100000 /lib/x86_64-linux-gnu/libpam.so.0
28512  sshd                3   0 02100000 /lib/x86_64-linux-gnu/libselinux.so.1
28512  sshd                3   0 02100000 /lib/x86_64-linux-gnu/libsystemd.so.0
28512  sshd                3   0 02100000 /usr/lib/x86_64-linux-gnu/libcrypto.so.1.0.2
28512  sshd                3   0 02100000 /lib/x86_64-linux-gnu/libutil.so.1
```

-f 选项能基于 open(2) 调用的标志进行过滤, 比如:

```
# ./opensnoop -e -f O_WRONLY -f O_RDWR
PID    COMM               FD ERR FLAGS    PATH
28084  clear_console       3   0 00100002 /dev/tty
28084  clear_console      -1  13 00100002 /dev/tty0
28084  clear_console      -1  13 00100001 /dev/tty0
28084  clear_console      -1  13 00100002 /dev/console
28084  clear_console      -1  13 00100001 /dev/console
28051  sshd                8   0 02100002 /var/run/utmp
28051  sshd                7   0 00100001 /var/log/wtmp
```


## 基于 cgroup 集进行过滤

--cgroupmap 选项基于 cgroup 集进行过滤, 它用于使用外部创建的映射.

```
# ./opensnoop --cgroupmap /sys/fs/bpf/test01
```

更多信息, 查看 [docs/special_filtering.md](https://github.com/iovisor/bcc/blob/master/docs/special_filtering.md)


## USAGE 说明

```
# ./opensnoop -h
usage: opensnoop.py [-h] [-T] [-U] [-x] [-p PID] [-t TID]
                    [--cgroupmap CGROUPMAP] [--mntnsmap MNTNSMAP] [-u UID]
                    [-d DURATION] [-n NAME] [-e] [-f FLAG_FILTER]

跟踪 open() 系统调用

optional arguments:
  -h, --help            show this help message and exit
  -T, --timestamp       include timestamp on output
  -U, --print-uid       include UID on output
  -x, --failed          only show failed opens
  -p PID, --pid PID     trace this PID only
  -t TID, --tid TID     trace this TID only
  --cgroupmap CGROUPMAP
                        trace cgroups in this BPF map only
  --mntnsmap MNTNSMAP   trace mount namespaces in this BPF map on
  -u UID, --uid UID     trace this UID only
  -d DURATION, --duration DURATION
                        total duration of trace in seconds
  -n NAME, --name NAME  only print process names containing this name
  -e, --extended_fields
                        show extended fields
  -f FLAG_FILTER, --flag_filter FLAG_FILTER
                        filter on flags argument (e.g., O_WRONLY)

examples:
    ./opensnoop           # trace all open() syscalls
    ./opensnoop -T        # include timestamps
    ./opensnoop -U        # include UID
    ./opensnoop -x        # only show failed opens
    ./opensnoop -p 181    # only trace PID 181
    ./opensnoop -t 123    # only trace TID 123
    ./opensnoop -u 1000   # only trace UID 1000
    ./opensnoop -d 10     # trace for 10 seconds only
    ./opensnoop -n main   # only print process names containing "main"
    ./opensnoop -e        # show extended fields
    ./opensnoop -f O_WRONLY -f O_RDWR  # only print calls for writing
    ./opensnoop --cgroupmap mappath  # only trace cgroups in this BPF map
    ./opensnoop --mntnsmap mappath   # only trace mount namespaces in the map
```
