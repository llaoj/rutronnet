---
title: "[解决] Warning pod/calico-node-<hash> Readiness probe failed"
date: "2022-03-21"
summary: "calico-node-4fpgp Readiness probe failed, orphaned pod <pod-hash> found, but volume paths are still present on disk : There were a total of N errors similar to this. Turn up verbosity to see them."
menu: "main"
tags:
- "kubelet"
- "kubernetes"
categories:
- "technology"
---


## 现象

开始发现 calico-system 名称空间下一直有一个 warning 的事件:

```sh
$ kubectl -n calico-system get events
LAST SEEN   TYPE      REASON      OBJECT                  MESSAGE
6m1s        Warning   Unhealthy   pod/calico-node-4fpgp   Readiness probe failed:
```

随后发现 kubelet 日志, 一直在报有两个孤立的 pod.

```sh
$ tail -f /var/log/messages
Mar 20 13:20:37 pcosmo-hda-ceno-06 kubelet: E0320 13:20:37.754807   20121 kubelet_volumes.go:154] orphaned pod "47e20a85-99ba-4c77-9cac-36d1aa56b6d3" found, but volume paths are still present on disk : There were a total of 2 errors similar to this. Turn up verbosity to see them.
```

## 解决办法

手动清理主机上遗留的 pod 文件夹:

```sh
$ rm -rf /var/lib/kubelet/pods/ce3bccb8-560d-4cda-b054-bf138291aa42/
```

## 原因分析

孤立的 pod 没有正确清理和 calico-node 就绪检查失败应该没有直接的关联, 大概是 kubelet 就绪检查受到 kubelet_volumes.go 影响.