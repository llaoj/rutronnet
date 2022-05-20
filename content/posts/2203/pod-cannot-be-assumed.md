---
title: "[解决] FailedScheduling pod/<pod-name> pod is <uid> in the cache so can't be assumed"
summary: "pod is in the cache, so can't be assumed, 这是调度器 scheduler 缓存失效导致的异常事件, 大致原因是 pod 已经调度, 并绑定到指定节点, 由于该节点异常导致启动失败, 重新启动 prometheus statefulset, 让集群重新调度, 其实就是将现有到 prometheus pod 副本数将至 0, 再恢复正常即可."
date: "2022-03-21"
menu: "main"
tags:
- "kubernetes"
categories:
- "technology"
---

## 现象&分析

之前由于 prometheus 所在节点 nfs 异常, 更换 prometheus 存储类型 (local-path) 并重新调度 prometheus 之后, 发现集群出现异常事件:

```sh
$ kubectl -n monitoring get  events
LAST SEEN   TYPE      REASON             OBJECT                 MESSAGE
60m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
58m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
57m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
55m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
54m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
52m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
51m         Warning   FailedScheduling   pod/prometheus-k8s-1   pod 19b50126-c636-4d3c-842e-768e76e3357b is in the cache, so can't be assumed
```

参考[#56682](https://github.com/kubernetes/kubernetes/issues/56682), 这是调度器 scheduler 缓存失效导致的异常事件, 大致原因是 pod 已经调度, 并绑定到指定节点, 由于该节点异常导致启动失败.

## 解决办法

重新启动 prometheus statefulset, 让集群重新调度, 其实就是将现有到 prometheus pod 副本数将至 0, 再恢复正常即可:

```sh
# prometheus crd 是 prometheus-operator 的自定义资源
# 用来定义部署配置
kubectl -n monitoring edit prometheus k8s
# 先 replicas => 0
:wq
# 再 replicas => 2
:wq
```

## 变更操作影响

届时 prometheus 会短暂(几分钟)停止服务, 旧数据无丢失.