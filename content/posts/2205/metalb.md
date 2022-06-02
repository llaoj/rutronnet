---
title: "MetalLB"
description: ""
summary: ""
date: "2022-05-30"
menu: "main"
tags:
- kubernetes
categories:
- "technology"
---

[参考文章](https://metallb.universe.tf/)

## 概念

### 使用BGP模式

在 BGP 模式下，集群中的每个节点都会与您的网络路由器建立 BGP 对等会话，并使用该对等会话来通告外部集群服务的 IP。  
假设您的路由器配置为支持多路径，这将实现真正的负载平衡：MetalLB 发布的路由彼此等效。这意味着路由器将使用所有下一跳，并在它们之间进行负载平衡。  
数据包到达节点后，kube-proxy 负责流量路由的最后一跳，将数据包送到服务中的特定 pod。

#### 负载均衡行为

负载平衡的确切行为取决于您的特定路由器型号和配置，但常见的行为是基于 `packet hash` 对 `per-connection` 进行平衡。这是什么意思？  
`per-connection`意味着单个 TCP 或 UDP 会话的所有数据包将被定向到集群中的单个机器。流量传播只发生在不同的连接之间，而不是一个连接内的数据包。  
这是一件好事，因为在多个集群节点上传播数据包会导致一些问题：  
- 