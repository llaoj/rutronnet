---
title: "bcc 之 tcplife 工具的使用"
description: "bcc 之 tcplife 工具的使用文档 "
summary: "这篇文档主要演示了 tcplife(Linux eBPF/bcc) 工具的使用. tcplife 总结了在跟踪期间打开和关闭的 TCP 会话. 比如"
date: "2022-03-16"
menu: "main"
tags:
- "bcc"
- "ebpf"
categories:
- "technology"
---

这篇文档主要演示了 tcplife(Linux eBPF/bcc) 工具的使用.


## 示例

tcplife 总结了在跟踪期间打开和关闭的 TCP 会话. 比如:

```
# ./tcplife
PID   COMM       LADDR           LPORT RADDR           RPORT TX_KB RX_KB MS
22597 recordProg 127.0.0.1       46644 127.0.0.1       28527     0     0 0.23
3277  redis-serv 127.0.0.1       28527 127.0.0.1       46644     0     0 0.28
22598 curl       100.66.3.172    61620 52.205.89.26    80        0     1 91.79
22604 curl       100.66.3.172    44400 52.204.43.121   80        0     1 121.38
22624 recordProg 127.0.0.1       46648 127.0.0.1       28527     0     0 0.22
3277  redis-serv 127.0.0.1       28527 127.0.0.1       46648     0     0 0.27
22647 recordProg 127.0.0.1       46650 127.0.0.1       28527     0     0 0.21
3277  redis-serv 127.0.0.1       28527 127.0.0.1       46650     0     0 0.26
[...]
```

这捕获了一个程序 “recordProg”, 它建立了一些到 “redis-serv” 的短暂的 TCP 连接, 每个连接持续大约 0.25 毫秒. 还有几个 “curl” 会话也被跟踪, 连接到端口 80, 持续了 91 和 121 毫秒.

此工具对于工作负载表征和流量统计很有用: **识别正在发生的连接以及传输的字节**.


在这个例子中, 我上传了一个 10 Mbyte 的文件到服务器, 然后再次使用 scp 下载:

```
# ./tcplife
PID   COMM       LADDR           LPORT RADDR           RPORT TX_KB RX_KB MS
7715  recordProg 127.0.0.1       50894 127.0.0.1       28527     0     0 0.25
3277  redis-serv 127.0.0.1       28527 127.0.0.1       50894     0     0 0.30
7619  sshd       100.66.3.172    22    100.127.64.230  63033     5 10255 3066.79
7770  recordProg 127.0.0.1       50896 127.0.0.1       28527     0     0 0.20
3277  redis-serv 127.0.0.1       28527 127.0.0.1       50896     0     0 0.24
7793  recordProg 127.0.0.1       50898 127.0.0.1       28527     0     0 0.23
3277  redis-serv 127.0.0.1       28527 127.0.0.1       50898     0     0 0.27
7847  recordProg 127.0.0.1       50900 127.0.0.1       28527     0     0 0.24
3277  redis-serv 127.0.0.1       28527 127.0.0.1       50900     0     0 0.29
7870  recordProg 127.0.0.1       50902 127.0.0.1       28527     0     0 0.29
3277  redis-serv 127.0.0.1       28527 127.0.0.1       50902     0     0 0.30
7798  sshd       100.66.3.172    22    100.127.64.230  64925 10265     6 2176.15
[...]
```

可以看到 sshd 接收了 10 MB, 然后再传输出去. 看起来, 接收(3.07 秒)比传输(2.18 秒)慢.


## 宽显

进程名称被截断为 10 个字符. 通过使用宽选项 -w, 列宽变为 16 个字符. IP 地址栏也更宽, 以适合 IPv6 地址:

```
# ./tcplife -w
PID   COMM             IP LADDR                      LPORT RADDR                      RPORT  TX_KB  RX_KB MS
26315 recordProgramSt  4  127.0.0.1                  44188 127.0.0.1                  28527      0      0 0.21
3277  redis-server     4  127.0.0.1                  28527 127.0.0.1                  44188      0      0 0.26
26320 ssh              6  fe80::8a3:9dff:fed5:6b19   22440 fe80::8a3:9dff:fed5:6b19   22         1      1 457.52
26321 sshd             6  fe80::8a3:9dff:fed5:6b19   22    fe80::8a3:9dff:fed5:6b19   22440      1      1 458.69
26341 recordProgramSt  4  127.0.0.1                  44192 127.0.0.1                  28527      0      0 0.27
3277  redis-server     4  127.0.0.1                  28527 127.0.0.1                  44192      0      0 0.32
```


## 添加时间戳

可以使用 -t 添加时间戳:

```
# ./tcplife -t
TIME(s)   PID   COMM       LADDR           LPORT RADDR           RPORT TX_KB RX_KB MS
0.000000  5973  recordProg 127.0.0.1       47986 127.0.0.1       28527     0     0 0.25
0.000059  3277  redis-serv 127.0.0.1       28527 127.0.0.1       47986     0     0 0.29
1.022454  5996  recordProg 127.0.0.1       47988 127.0.0.1       28527     0     0 0.23
1.022513  3277  redis-serv 127.0.0.1       28527 127.0.0.1       47988     0     0 0.27
2.044868  6019  recordProg 127.0.0.1       47990 127.0.0.1       28527     0     0 0.24
2.044924  3277  redis-serv 127.0.0.1       28527 127.0.0.1       47990     0     0 0.28
3.069136  6042  recordProg 127.0.0.1       47992 127.0.0.1       28527     0     0 0.22
3.069204  3277  redis-serv 127.0.0.1       28527 127.0.0.1       47992     0     0 0.28
```

这表明 recordProg 进程每秒连接一次.

另外 -T 选项可以使用 HH:MM:SS 格式时间戳.


## 逗号分隔列表

-s 选项可以指定逗号分隔的列表模式. 这里同时使用了 -t 和 -T 类型的时间戳:

```
# ./tcplife -stT
TIME,TIME(s),PID,COMM,IP,LADDR,LPORT,RADDR,RPORT,TX_KB,RX_KB,MS
23:39:38,0.000000,7335,recordProgramSt,4,127.0.0.1,48098,127.0.0.1,28527,0,0,0.26
23:39:38,0.000064,3277,redis-server,4,127.0.0.1,28527,127.0.0.1,48098,0,0,0.32
23:39:39,1.025078,7358,recordProgramSt,4,127.0.0.1,48100,127.0.0.1,28527,0,0,0.25
23:39:39,1.025141,3277,redis-server,4,127.0.0.1,28527,127.0.0.1,48100,0,0,0.30
23:39:41,2.040949,7381,recordProgramSt,4,127.0.0.1,48102,127.0.0.1,28527,0,0,0.24
23:39:41,2.041011,3277,redis-server,4,127.0.0.1,28527,127.0.0.1,48102,0,0,0.29
23:39:42,3.067848,7404,recordProgramSt,4,127.0.0.1,48104,127.0.0.1,28527,0,0,0.30
23:39:42,3.067914,3277,redis-server,4,127.0.0.1,28527,127.0.0.1,48104,0,0,0.35
[...]
```

## 端口过滤

还有过滤本地/远端端口的选项. 这里就过滤了本地 22 和 80 端口.

```
# ./tcplife.py -L 22,80
PID   COMM       LADDR           LPORT RADDR           RPORT TX_KB RX_KB MS
8301  sshd       100.66.3.172    22    100.127.64.230  58671     3     3 1448.52
[...]
```


## USAGE 说明

```
# ./tcplife.py -h
usage: tcplife.py [-h] [-T] [-t] [-w] [-s] [-p PID] [-L LOCALPORT]
                  [-D REMOTEPORT] [-4 | -6]

跟踪 TCP 会话的生命周期并进行总结

optional arguments:
  -h, --help            show this help message and exit
  -T, --time            include time column on output (HH:MM:SS)
  -t, --timestamp       include timestamp on output (seconds)
  -w, --wide            wide column output (fits IPv6 addresses)
  -s, --csv             comma separated values output
  -p PID, --pid PID     trace this PID only
  -L LOCALPORT, --localport LOCALPORT
                        comma-separated list of local ports to trace.
  -D REMOTEPORT, --remoteport REMOTEPORT
                        comma-separated list of remote ports to trace.
  -4, --ipv4            trace IPv4 family only
  -6, --ipv6            trace IPv6 family only

examples:
    ./tcplife           # trace all TCP connect()s
    ./tcplife -t        # include time column (HH:MM:SS)
    ./tcplife -w        # wider columns (fit IPv6)
    ./tcplife -stT      # csv output, with times & timestamps
    ./tcplife -p 181    # only trace PID 181
    ./tcplife -L 80     # only trace local port 80
    ./tcplife -L 80,81  # only trace local ports 80 and 81
    ./tcplife -D 80     # only trace remote port 80
    ./tcplife -4        # only trace IPv4 family
    ./tcplife -6        # only trace IPv6 family
```