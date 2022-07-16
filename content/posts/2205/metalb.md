---
title: "MetalLB"
description: ""
summary: ""
date: "2022-05-30"
menu: "main"
draft: true
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

负载均衡的确切行为取决于您的特定路由器型号和配置，但常见的行为是基于 `packet-hash` 对 `per-connection` 进行平衡。这是什么意思？  
`per-connection`意味着单个 TCP 或 UDP 会话的**所有数据包**将被定向到集群中的单个机器。流量传播只发生在不同的连接之间，而不是一个连接内的数据包。  
这是一件好事，因为在多个集群节点上传播数据包会导致一些问题：  

- 跨多个传输路径传播单个连接会导致数据包(packet)在线路上重新排序，这会极大地影响终端主机的性能。
- 在 Kubernetes 中, 不能保证节点之间流量的路由保持一致。这意味着两个不同的节点可以将同一连接的数据包(packet)路由到不同的 Pod，这将导致连接失败。

> 一个连接(connection)由多个连续的数据包(packet)构成

Packet hashing is how high-performance routers can statelessly spread connections across multiple backends. For each packet, they extract some of the fields, and use those as a “seed” to deterministically pick one of the possible backends. If all the fields are the same, the same backend will be chosen.

The exact hashing methods available depend on the router hardware and software. Two typical options are 3-tuple and 5-tuple hashing. 3-tuple uses (protocol, source-ip, dest-ip) as the key, meaning that all packets between two unique IPs will go to the same backend. 5-tuple hashing adds the source and destination ports to the mix, which allows different connections from the same clients to be spread around the cluster.

In general, it’s preferable to put as much entropy as possible into the packet hash, meaning that using more fields is generally good. This is because increased entropy brings us closer to the “ideal” load-balancing state, where every node receives exactly the same number of packets. We can never achieve that ideal state because of the problems we listed above, but what we can do is try and spread connections as evenly as possible, to try and prevent hotspots from forming.

#### 局限性

Using BGP as a load-balancing mechanism has the advantage that you can use standard router hardware, rather than bespoke load balancers. However, this comes with downsides as well.

The biggest downside is that BGP-based load balancing does not react gracefully to changes in the backend set for an address. What this means is that when a cluster node goes down, you should expect all active connections to your service to be broken (users will see “Connection reset by peer”).

BGP-based routers implement stateless load balancing. They assign a given packet to a specific next hop by hashing some fields in the packet header, and using that hash as an index into the array of available backends.

The problem is that the hashes used in routers are usually not stable, so whenever the size of the backend set changes (for example when a node’s BGP session goes down), existing connections will be rehashed effectively randomly, which means that the majority of existing connections will end up suddenly being forwarded to a different backend, one that has no knowledge of the connection in question.

The consequence of this is that any time the IP→Node mapping changes for your service, you should expect to see a one-time hit where most active connections to the service break. There’s no ongoing packet loss or blackholing, just a one-time clean break.

Depending on what your services do, there are a couple of mitigation strategies you can employ:

Your BGP routers might have an option to use a more stable ECMP hashing algorithm. This is sometimes called “resilient ECMP” or “resilient LAG”. Using such an algorithm hugely reduces the number of affected connections when the backend set changes.
Pin your service deployments to specific nodes, to minimize the pool of nodes that you have to be “careful” about.
Schedule changes to your service deployments during “trough”, when most of your users are asleep and your traffic is low.
Split each logical service into two Kubernetes services with different IPs, and use DNS to gracefully migrate user traffic from one to the other prior to disrupting the “drained” service.
Add transparent retry logic on the client side, to gracefully recover from sudden disconnections. This works especially well if your clients are things like mobile apps or rich single-page web apps.
Put your services behind an ingress controller. The ingress controller itself can use MetalLB to receive traffic, but having a stateful layer between BGP and your services means you can change your services without concern. You only have to be careful when changing the deployment of the ingress controller itself (e.g. when adding more NGINX pods to scale up).
Accept that there will be occasional bursts of reset connections. For low-availability internal services, this may be acceptable as-is.

#### FRR 模式

MetalLB 提供了一个实验模式: 使用 FRR 作为 BGP 层的后端.

开启 FRR 模式之后, 会获得以下额外的特性:

- BFD 支持的 BGP 会话
- BGP 和 BFD 支持 IPv6

#### FRR 模式的局限性

Compared to the native implementation, the FRR mode has the following limitations:

- The RouterID field of the BGPAdvertisement can be overridden, but it must be the same for all the advertisements (there can’t be different advertisements with different RouterIDs).
- The myAsn field of the BGPAdvertisement can be overridden, but it must be the same for all the advertisements (there can’t be different advertisements with different myAsn).
- In case a eBGP Peer is multiple hops away from the nodes, the ebgp-multihop flag must be set to true.

