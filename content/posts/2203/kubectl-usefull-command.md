---
title: "比较冷门但有用的 kubectl 命令"
description: "比较冷门但有用的 kubectl 命令 "
summary: "以下冷门命令能实现某种具体的功能, 都是在实际工作中摸索总结的经验, 获取到相关的资源名称之后, 就可以配合常用的 kubectl 命令获取其他详细信息."
date: "2022-03-22"
menu: "main"
tags:
- "kubectl"
- "kubernetes"
categories:
- "技术"
---

以下冷门命令能实现某种具体的功能, 都是在实际工作中摸索总结的经验, 获取到相关的资源名称之后, 就可以配合常用的 kubectl 命令获取其他详细信息.

## 列出所有被 OOMKilled 过的 POD 列表

使用 [jq](https://stedolan.github.io/jq/) (一个轻量级的灵活的命令行JSON解析器) 获取上一个状态

```sh
kubectl get pods -A -ojson | jq -c '.items[] | {name: .metadata.name,reasons: [{reason: .status.containerStatuses[]?.lastState.terminated.reason, finishedAt: .status.containerStatuses[]?.lastState.terminated.finishedAt}]}' | grep OOMKilled
```

pod 信息以行为单位显示, 方便进行筛选:

```sh
# 按行输出
{"name":"nginx-699f949679-8kkth","reasons":[{"reason":"OOMKilled","fini  
shedAt":"2021-09-08T01:34:05Z"}]}
...
```


## 列出含有 UID 的 POD 列表

kubelet 大量使用了 pod-uid 来为 pod 创建相关的资源, 获取 pod-uid 来辅助运维是很有必要的.

```sh
kubectl get pods -A -o custom-columns=NAME:.metadata.name,UID:.metadata.uid
```

