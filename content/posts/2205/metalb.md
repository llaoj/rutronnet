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

[官方文档](https://metallb.universe.tf/)

## 为什么使用?

Kubernetes没有提供裸适用于金属集群的网络负载均衡器实现, 也就是LoadBalancer类型的Service. Kubernetes 附带的网络负载均衡器的实现都是调用各种 IaaS 平台（GCP、AWS、Azure ……）的胶水代码。 如果您没有在受支持的 IaaS 平台（GCP、AWS、Azure...）上运行，LoadBalancers 在创建时将一直保持在`pending`状态。

裸金属集群的运维人员只剩下两个较小的工具来将用户流量引入集群内: `NodePort`和`externalIPs`类型Service. 这两种在生产环境使用有很大的缺点, 这让裸金属集群成为Kubernetes生态中的第二类选择, 并不是首选.

MetalLB的目的是实现一个网络负载均衡器来与标准的网络设备集成, 这样外部服务就可以尽可能的正常工作了.

## 要求

MetalLB 要求如下:


- 一个 Kubernetes 集群, Kubernetes 版本 1.13.0+, 没有网络负载均衡器功能.
- 可以与 MetalLB 共存的集群网络配置。
- 一些供 MetalLB 分发的 IPv4 地址。
- 当使用 BGP 操作模式时，您将需要一台或多台能够广播 BGP 的路由器。
- 使用 L2 操作模式时，节点之间必须允许 7946 端口（TCP 和 UDP，可配置其他端口）上的流量，这是 [memberlist](https://github.com/hashicorp/memberlist) 的要求。

## 功能

MetalLB 是作为 Kubernetes 中的一个组件, 提供了一个中网络负载均衡器的实现. 简单来说, 在非公有云环境搭建的集群上, 不能使用公有云的负载均衡器, 它可以让你在集群中创建 LoadBalancer 类型的 Service.

为了提供这样的服务, 它具备两个功能: 地址分配、对外发布

### 地址分配

在公有云的 Kubernetes 集群中, 你申请一个负载均衡器, 云平台会给你分配一个 IP 地址. 在裸金属集群中, MetalLB 来做地址分配.

MetalLB 不能凭空造 IP, 所以你需要提供供它使用的 IP 地址池. 在 Service 的创建和删除过程中, MetalLB 会对 Service 分配和回收 IP, 这些都是你配置的地址池中的 IP.

如何获取 MetalLB 的 IP 地址池取决于您的环境。 如果您在托管设施中运行裸机集群，您的托管服务提供商可能会提供 IP 地址供出租。 在这种情况下，您将租用例如 /26 的 IP 空间（64 个地址），并将该范围提供给 MetalLB 以用于集群服务。

同样, 如果你的集群是纯私有的, 可以提供一个没有暴露到网络中的相邻的 LAN 网段. 这种情况下, 你可以抽取私有地址空间( RFC1918 地址)中的一段 IPs 地址, 分配给 MetalLB. 这种地址是免费的, 只要你只把它提供给当前 LAN 内的集群 Services, 它们就能正常工作.

或者你可以两者都用! MetalLB 可以让你定义多个地址池, 不需要关心这些地址的`类别`.

### 对外发布

当 MetalLB 给 Service 分配一个对外使用的IP之后, 它需要让集群所在的网络知道这个 IP 的存在. MetalLB 使用标准的网络/路由协议来实现, 这要看具体的工作模式: ARP、NDP 或者 BGP.

#### 2层模式 (ARP/NDP)

在2层模式下, 集群中的机器使用标准地址发现协议把这些 IPs 通知本地网络. 从 LAN 的角度看, 这台服务器有多个 IP 地址. 这个模式的详细行为还有局限性下面会介绍.

#### BGP

在 BGP 模式下, 集群中的所有机器与临近的路由器建立 BGP 对等会话, 告诉他们如何路由 Service IPs. 得益于 BGP 的策略机制, 使用 BGP 能实现真正的多节点负载均衡和细粒度的流量控制. 下面会介绍更多操作和局限性方面的细节.

## 工作模式

### 2层模式

在2层工作模式下, 一个节点负责向本地网络发布服务. 从网络的角度看, 更像是这台服务器的网卡有多个 IP 地址. 在底层, MetalLB 会响应适用于 IPv4 的 ARP 请求和适用于 IPv6 的 NDP 请求. 这个工作模式的最大的优势就是适应性强: 它能工作在任何以太网内, 没有特殊的硬件要求, 不需要花哨的路由器.

#### 负载均衡行为

在2层模式下, 一个 Service IP 的流量都会到一个节点上. 在该节点上, `kube-proxy` 将流量分发到服务具体的 pods 上. L2 并没有实现负载均衡. 但是, 它实现了一个错误转移机制, 这样当领袖节点故障之后, 另一个节点就会接管这些 IP 地址. 如果领袖节点因为某些原因故障了, 故障转移是自动的: 使用 [memberlist](https://github.com/hashicorp/memberlist) 检测到节点发生故障, 同时新的节点会从故障节点上接管这些 IP.

#### 局限性

L2模式有两个主要局限性: 单点的瓶颈、潜在的缓慢故障转移.

如上面说的, 在 L2 模式下, 一个被选举的单一的领袖节点会接收所有的 Service IPs 的流量. 这意味着, 你服务的入口带宽受限与这个单一节点的带宽. 如果使用 ARP/NDP 引导流量, 这是一个基本限制.

当前的实现, 节点之间的故障转移依赖客户端之间的配合. 当故障发生时, MetalLB 会不经请求的发送出一些2层的数据包, 来通知其他客户端 Service IP 所对应的 MAC 地址已经更改.

大多数操作系统能正确处理这种数据包, 同时更新“邻居”的缓存. 这种情况下, 故障转移也就几秒钟. 但是, 有一些系统要么没有实现这种报文的处理, 要么实现了, 但是更新缓存很慢.

所有现代版本的操作系统都正确实现了 L2 故障转移, 比如 Windows、Mac、Linux. 所以, 出问题的仅仅是很老或者不常见的操作系统.

为了最大限度地减少计划内的故障转移对有故障的客户端的影响，您应该让旧的领袖节点多运行几分从, 以便它可以继续为旧客户端转发流量，直到它们的缓存刷新。

当一个计划之外的故障出现时. 在访问出错的客户端刷新它们的缓存之前, Service IPs 将不可达.

#### 和 keepalive 比较

MetalLB 的2层模式和 keepalived 有很多相似之处. 所以, 如果你熟悉 keepalived, 我说的很多你应该很熟悉. 但是和它也有一些不同的地方需要说一下.

Keepalived 使用虚拟路由器冗余协议(VRRP). Keepalived 的各实例之间不断地相互交换 VRRP 消息，以选择领袖并监控该领导者何时离开。

但是, MetalLB 却是依赖 memberlist 项目来知道什么时候集群中的节点不可达, 什么时候这个节点的 Service IPs 需要移动到别处.

Keepalived 和 MetalLB 从客户端的角度“看起来”是一样的：当发生故障转移时，Service IP 地址从一台机器迁移到另一台机器，之后该机器就会有多个 IP 地址。

因为它不用 VRRP, MetalLB 并不会有这个协议本身的局限性. 比如: VRRP协议限制每个网络只能用255个负载负载均衡器, 但是 MetalLB 就没有这个限制. 你可以有很多负载均衡的 IPs, 只要你网络中有空闲 IPs. MetalLB 的配置比 Keepalived 少, 比如, 它不需要 `Virtual Router IDs`.

另一方面，由于 MetalLB 依赖于 memberlist 来获取集群成员信息，它无法与第三方 VRRP 感知路由器和基础设施进行互操作。 这是设计之初规定好的: MetalLB 专门设计**用于在 Kubernetes 集群内**提供负载平衡和故障转移.

### BGP模式

在 BGP 模式下，集群中的每个节点都会与您的网络路由器建立 BGP 对等会话，并使用该对等会话来通告外部集群服务的 IP.

假设您的路由器配置为支持多路径，这将实现真正的负载平衡: MetalLB 发布的路由彼此等效。这意味着路由器将使用所有下一跳，并在它们之间进行负载平衡.

数据包到达节点后，kube-proxy 负责流量路由的最后一跳，将数据包送到服务中的特定 pod.

#### 负载均衡行为

负载均衡的确切行为取决于您的特定路由器型号和配置，但常见的行为是基于 `packet-hash` 方法在连接层面 `per-connection` 进行平衡。这是什么意思?

这个`per-connection`意味着单个 TCP 或 UDP 会话的**所有数据包**将被定向到集群中的单个机器。流量传播只发生在不同的连接之间，而不是一个连接内的数据包. 这是一件好事，因为在多个集群节点上传播数据包会导致一些问题：  

- 跨多个传输路径传播单个连接会导致数据包(packet)在线路上重新排序，这会极大地影响终端主机的性能。
- 在 Kubernetes 中, 不能保证节点之间流量的路由保持一致。这意味着两个不同的节点可以将同一连接的数据包(packet)路由到不同的 Pod，这将导致连接失败。

高性能路由器能够以一种无状态的方式在多个后端之间使用数据包哈希的方法散布数据包. 对于每一个数据包, 它们拥有一些属性, 并能用它作为 “种子” 来决定选择哪一个后端. 如果, 所有的属性都一样, 它们就会选择同一个后端. 

具体使用哪种哈希方法取决于路由器的硬件和软件. 典型的方法是: `3-tuple` 和 `5-tuple`. 3-tuple 哈希法使用数据包中的协议、源IP、目的IP作为哈希键, 这意味着来自不同ip的数据包会进入同一个后端. 5-tuple 哈希法又在其中加入了源端口和目的端口, 所有来自相同客户端的不同连接将会在集群中散布. 

通常, 我们推荐加入尽可能多的属性来参与数据包哈希, 也就是说使用更多的属性是非常好的. 因为这样会更加接近理想的负载均衡状态, 每一个节点都会收到相同数量的数据包. 但是我们永远不会达到这种理想状态, 因为上述原因, 但是我们能做的就是尽可能的均匀的传播连接, 以防止出现主机热点.

> 一个连接(connection)由多个连续的数据包(packet)构成

#### 局限性

使用 BGP 作为负载均衡机制可以让你使用标准的路由器硬件, 而不是定制的负载均衡器. 但是, 这也带来的一些缺点:

最大的缺点是基于 BGP 的负载平衡不能优雅地响应后端设置的地址更改. 也就是说, 当集群的一个节点下线了, 到你服务的所有成功的连接都会损坏(用户会看到报错:`Connection reset by peer`)

基于 BGP 的路由器实现无状态负载均衡。 他们通过哈希数据包头中的一些字段并将该哈希用作可用后端数组的索引，将给定数据包分配给特定的下一跳。

问题是路由器中使用的哈希值通常*不稳定*，所以每当后端集的大小发生变化时（例如，当一个节点的 BGP 会话关闭时），现有的连接将被有效地随机重新哈希，这意味着大多数现有的 连接最终会突然被转发到不同的后端，一个不知道相关连接的后端。

结果是每当你的服务的 `IP > Node` 映射发生变化时, 你希望看到一次性干净的切换出现, 到该服务的大部分所有可用连接中断. 没有持续的丢包或者黑洞, 只是一次很干净的中断而已.

根据您的服务的用途，您可以采用几种缓解策略：

- 你的 BGP 路由器可能有更加稳定的ECMP哈希算法. 有时候可能叫: `弹性ECMP` 或 `弹性LAG`. 使用这样的算法, 在后端集合发生变化的时候, 能有效的减少受影响的连接数量.
- 把你的服务部署到特定的节点上, 来减小节点池的大小, 它需要你小心的照顾.
- 把服务的变更安排在流量的低谷时候, 此时你的用户在睡觉流量很低.
- 把每一个逻辑上的服务拆分成两个有着不同 IP 的 kubernetes 服务, 使用DNS服务优雅的将用户流量从将要中断的服务迁移到另一个服务上.
- 在客户端添加重试逻辑, 以优雅地从突然断开连接中恢复. 如果您的客户是移动应用程序或丰富的单页网络应用程序, 这尤其适用.
- 将您的服务放在 ingress 控制器后面. ingress 控制器本身可以使用 MetalLB 来接收流量, 但是在 BGP 和您的服务之间有一个状态层意味着您可以毫无顾虑地更改您的服务。 您只需在更改 ingress 控制器本身的部署时小心(例如, 在添加更多 NGINX pod 时).
- 接受偶尔会出现重置连接的情况. 对于低可用性的内部服务, 这可能是可以接受的.

#### FRR 模式

MetalLB 提供了一个实验模式: 使用 FRR 作为 BGP 层的后端.

开启 FRR 模式之后, 会获得以下额外的特性:

- BFD 支持的 BGP 会话
- BGP 和 BFD 支持 IPv6

#### FRR 模式的局限性

相比与原生实现, FRR 模式有以下局限性:

- BGPAdvertisement 的 RouterID 字段可以被覆盖，但它必须对所有的 advertisements 都相同（不能有不同的 advertisements 具有不同的 RouterID）。
- BGPAdvertisement 的 myAsn 字段可以被覆盖，但它必须对所有 advertisements 都相同(不能有不同的 advertisements 具有不同的 myAsn)
- 如果 eBGP Peer 是距离节点多跳的, 则 ebgp-multihop 标志必须设置为 true

## 安装

安装之前, 确保满足所有[要求](#要求). 尤其是, 你要注意[网络附加组件的兼容性](#网络附加组件的兼容性)

如果你在云平台环境运行 MetalLB, 你最好先看看[云环境兼容性](https://metallb.universe.tf/installation/clouds/)页面, 确保你选择的云平台可以和 MetalLB 一起正常工作(大多数情况下都不好用).

MetalLB 支持4种安装方式: 

- 使用 Kubernetes 部署清单安装
- 使用 Kustomize 安装
- 使用 Helm 安装
- 使用 MetalLB Operator 安装

### 准备

如果您在 IPVS 模式下使用 kube-proxy，则从 Kubernetes v1.14.2 开始，您必须启用严格的 ARP 模式。

> 请注意，如果您使用 kube-router 作为服务代理，则不需要这个，因为它默认启用严格的 ARP。

你可以在集群中修改 kube-proxy 的配置文件:

```shell
kubectl edit configmap -n kube-system kube-proxy
```

这样配置:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

你也可以把这个配置片段加入到你的 `kubeadm-config` 中, 在主配置后面使用 `---` 附加上它就行.

如果你想自动更改配置, 下面的 shell 脚本可以用:

```shell
# 查看将会发生什么样的配置变化, 如果存在不同则返回非零返回值
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# 实际应用变更, 仅在遇到错误的时候返回非零返回值
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### 使用部署清单安装

使用下面的部署清单, 来安装 MetalLB:

```shell
kubectl apply -f https://raw.githubusercontent.com/
metallb/metallb/v0.13.4/config/manifests/metallb-native.yaml
```
> 如果你想开启使用实验性的FRR模式, 使用下面的部署清单
> `kubectl apply -f https://raw.githubusercontent.com`
> `/metallb/metallb/v0.13.4/config/manifests/metallb-frr.yaml`
> 注意: 这些部署清单来自开发分支. 如果应用在生产环境, 强烈推荐你使用稳定的发布版本.

这样, 在 `metallb-system` 名称空间下就部署了 MetalLB. 主要组件是:

- Deployment: `metallb-system/controller`, 这是集群级别的控制器, 负责处理 IP 分配.
- Daemonset: `metallb-system/speaker`, 这个组件使用你选择的协议对外发送信息, 使你的 Service 可以被访问.
- ServiceAccount: controller 和 speaker 所使用的, 同时配置好了组件所需要的 RBAC 权限.

该安装清单并不包含配置文件, 但是 MetalLB 组件仍会启动, 在你[部署配置相关资源](#配置)之前, 它保持空闲状态.

### 其他安装方式

另外的三种安装方式(Kustomize, Helm, MetalLB Operator)请查看[官方文档](https://metallb.universe.tf/installation/)

### 网络附加组件的兼容性

通常来说, MetalLB 并不关心你集群用什么网络附件组件, 只要它能满足 Kubernetes 要求的标准就可以.

下面是一些网络附加组件与 MetalLB 一起测试的结果, 可以供你参考. 列表是按照附加组件的字母先后顺序排序, 我并没有任何倾向性.

列表中没有的附加组件可能也可以正常工作, 只是我们没有测试.

|网络附加组件|兼容性|
|-|-|
|Antrea|可以 1.4和1.5版本测试通过|
|Calico|大部分可以 查看[与Calico一起部署](#与Calico一起部署)|
|Canal|可以|
|Cilium|可以|
|Flannel|可以|
|Kube-ovn|可以|
|Kube-router|大部分可以 [已知问题](https://metallb.universe.tf/configuration/kube-router/)|
|Weave Net|大部分可以 [已知问题](https://metallb.universe.tf/configuration/weave/)|

## 配置

配置之前 MetalLB 一直保持空闲状态. 想要配置 MetalLB, 需要在 MetalLB 所在的名称空间(通常是 `metallb-system`)部署很多和配置相关的资源.

当然, 如果你没有把 MetalLB 部署到 `metallb-system` 名称空间下, 你可能需要修改下面的配置清单.

### 给定义LoadBalancer类型的Services可分配的IPs地址

为了能给 Services 分配 IPs, MetalLB 通过 `IPAddressPool` 自定义资源来定义.

通过 `IPAddressPools` 定义好 IPs 地址池之后, MetalLB 就会使用这些地址给 Services 分配 IP.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.0/24
  - 192.168.9.1-192.168.9.5
  - fc00:f853:0ccd:e799::/124
```

可以同时定义多个 `IPAddressPools` 资源. 可以使用 CIDR 定义地址, 也可以使用范围区间定义, 也可以定义 IPv4 和 IPv6 地址用于分配.

### 对外公布 Service IPs

一旦给 Service 分配IPs之后, 它们必须要公布出去. 具体的配置要根据你想使用的协议来定, 下面会一一介绍:

*注意: 也可以同时将一个 service 同时通过 L2 和 BGP 对外公布.*

### 使用 L2 模式的配置

配置2层模式最简单, 在多数情况下, 你不需要任何关于协议的配置, 只配置IP地址就行.

使用2层模式**不需要将IP地址绑定到工作节点的网络接口上**. 它直接响应本地网络上的ARP请求，将机器的MAC地址提供给客户端。

为了公布来自 `IPAddressPool` 的 IP, `L2Advertisement` 资源必须与 `IPAddressPool` 资源一起使用.

比如, 下面的例子配置了 MetalLB 可以分配从 192.168.1.240 到 192.168.1.250 之间的地址, 并配置它使用2层模式:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

上面的例子, 在 `L2Advertisement` 中没有配置 `IPAddressPool` 选择器, 这样它能使用所有的 IP 地址.

所以, 为了只将一部分 `IPAddressPool`s 使用 L2 对外公布, 我们就需要声明一下(或者, 使用标签选择器). 

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

### 使用 BGP 模式的配置

需要告诉 MetalLB 如何与外部一个或多个 BGP 路由器建立会话.

所以, 需要给每一个 MetalLB 需要连接到路由器建一个 `BGPPeer` 资源.

为了能提供一个基础的 BGP 路由器配置和一个 IP 地址范围, 你需要定义4段信息.

- MetalLB 需要连接的路由器地址
- 路由器的AS编号
- MetalLB 使用的AS编号
- 使用CIDR前缀表示IP地址范围

比如给MetalLB分配的AS编号为64500, 并且将它连接到地址为 `10.0.0.1` 且AS编号为 64501 的路由器，您的配置将如下所示：

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: sample
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 64501
  peerAddress: 10.0.0.1
```

提供一个IP地址池 `IPAddressPool` 像这样:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
```

使用自定义资源`BGPAdvertisement`来配置MetalLB使用BGP来公布IPs:

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: example
  namespace: metallb-system
```

在`BGPAdvertisement`的定义中, 如果没有配置`IPAddressPool`选择器, 默认使用所有可用的IP地址池`IPAddressPools`

如果仅需要使用特定的IP地址池通过BGP进行地址公布, 则需要声明一个IP地址池列表`ipAddressPools`(或者使用标签选择器):

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

### 为 BGP 会话开启 BFD 支持

在实验的FRR模式下, BGP会话可以由BFD会话取代，从而能实现比单纯使用BGP更快的路径故障检测的能力.

想要开启BFD, 你必须定义个BFD配置`BFDProfile`, 并且和BGP会话对等方配置`BGPPeer`关联起来:

```yaml
apiVersion: metallb.io/v1beta1
kind: BFDProfile
metadata:
  name: testbfdprofile
  namespace: metallb-system
spec:
  receiveInterval: 380
  transmitInterval: 270
```

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: peersample
  namespace: metallb-system
spec:
  myASN: 64512
  peerASN: 64512
  peerAddress: 172.30.0.3
  bfdProfile: testbfdprofile
```

### 配置验证

部署时 MetalLB 会安装配置验证的钩子程序 `validatingwebhook`, 用来检查用户部署的自定义资源的有效性.

但是, 因为 MetalLB 的整体配置被分隔成很多分片, 不是所有的无效配置都能被钩子程序避免. 所以, 一旦无效的 MetalLB 配置资源被成功部署了, MetalLB 会忽略它, 并使用最新的有效配置.

未来的版本中, MetalLB 会把错误的配置暴露到 Kubernetes 资源中. 但是, 目前来说, 如果需要知道为什么配置没有生效, 就需要去查看一下控制器的日志.

### 更多高级配置

关于地址分配、BGP、L2、calico等相关的高级配置, 请参考[官方文档](https://metallb.universe.tf/configuration/_advanced_ipaddresspool_config/)

## 如何使用

After MetalLB is installed and configured, to expose a service externally, simply create it with spec.type set to LoadBalancer, and MetalLB will do the rest.

MetalLB attaches informational events to the services that it’s controlling. If your LoadBalancer is misbehaving, run kubectl describe service <service name> and check the event log.

### 请求特定的IPs

MetalLB 尊重`spec.loadBalancerIP`参数, 所以, 如果你想要使用一个特定的IP地址, 你可以通过配置这个参数来指定. 如果MetalLB没有你申请的IP地址, 或者如果你申请的IP地址已经被其他的服务占用了, 地址分配就会失败, MetalLB会记录一个Warning级别的事件, 你通过`kubectl describe service <service name>`就能看到.

MetalLB不仅支持`spec.loadBalancerIP`参数, 还支持一个自定义 annotation 参数: `metallb.universe.tf/loadBalancerIPs`. 对于有些双栈的 Service 需要多个IPs, 这个 annotation 也支持使用逗号分隔指定多个IPs

**请注意**: 在 kubernetes 的API中, `spec.LoadBalancerIP`参数未来计划会被废弃.

如果你想使用特定的类型的IP, 但是不在乎具体是什么地址, MetalLB同样也支持请求一个特定的IP地址池. 为能能使用特定的地址池, 你需要在 Service 中添加一个 annotation: `metallb.universe.tf/address-pool`, 来指定IP地址池的名称. 比如:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    metallb.universe.tf/address-pool: production-public-ips
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

### 流量策略

MetalLB understands and respects the service’s externalTrafficPolicy option, and implements different announcements modes depending on the policy and announcement protocol you select.

#### Layer2
When announcing in layer2 mode, one node in your cluster will attract traffic for the service IP. From there, the behavior depends on the selected traffic policy.

##### “Cluster”流量策略
With the default Cluster traffic policy, kube-proxy on the node that received the traffic does load balancing, and distributes the traffic to all the pods in your service.

This policy results in uniform traffic distribution across all pods in the service. However, kube-proxy will obscure the source IP address of the connection when it does load balancing, so your pod logs will show that external traffic appears to be coming from the service’s leader node.

##### “Local”流量策略
With the Local traffic policy, kube-proxy on the node that received the traffic sends it only to the service’s pod(s) that are on the same node. There is no “horizontal” traffic flow between nodes.

Because kube-proxy doesn’t need to send traffic between cluster nodes, your pods can see the real source IP address of incoming connections.

The downside of this policy is that incoming traffic only goes to some pods in the service. Pods that aren’t on the current leader node receive no traffic, they are just there as replicas in case a failover is needed.

#### BGP
When announcing over BGP, MetalLB respects the service’s externalTrafficPolicy option, and implements two different announcement modes depending on what policy you select. If you’re familiar with Google Cloud’s Kubernetes load balancers, you can probably skip this section: MetalLB’s behaviors and tradeoffs are identical.

##### “Cluster”流量策略
With the default Cluster traffic policy, every node in your cluster will attract traffic for the service IP. On each node, the traffic is subjected to a second layer of load balancing (provided by kube-proxy), which directs the traffic to individual pods.

This policy results in uniform traffic distribution across all nodes in your cluster, and across all pods in your service. However, it results in two layers of load balancing (one at the BGP router, one at kube-proxy on the nodes), which can cause inefficient traffic flows. For example, a particular user’s connection might be sent to node A by the BGP router, but then node A decides to send that connection to a pod running on node B.

The other downside of the “Cluster” policy is that kube-proxy will obscure the source IP address of the connection when it does its load balancing, so your pod logs will show that external traffic appears to be coming from your cluster’s nodes.

##### “Local”流量策略
With the Local traffic policy, nodes will only attract traffic if they are running one or more of the service’s pods locally. The BGP routers will load balance incoming traffic only across those nodes that are currently hosting the service. On each node, the traffic is forwarded only to local pods by kube-proxy, there is no “horizontal” traffic flow between nodes.

This policy provides the most efficient flow of traffic to your service. Furthermore, because kube-proxy doesn’t need to send traffic between cluster nodes, your pods can see the real source IP address of incoming connections.

The downside of this policy is that it treats each cluster node as one “unit” of load balancing, regardless of how many of the service’s pods are running on that node. This may result in traffic imbalances to your pods.

For example, if your service has 2 pods running on node A and one pod running on node B, the Local traffic policy will send 50% of the service’s traffic to each node. Node A will split the traffic it receives evenly between its two pods, so the final per-pod load distribution is 25% for each of node A’s pods, and 50% for node B’s pod. In contrast, if you used the Cluster traffic policy, each pod would receive 33% of the overall traffic.

In general, when using the Local traffic policy, it’s recommended to finely control the mapping of your pods to nodes, for example using node anti-affinity, so that an even traffic split across nodes translates to an even traffic split across pods.

In future, MetalLB might be able to overcome the downsides of the Local traffic policy, in which case it would be unconditionally the best mode to use with BGP announcements. See issue 1 for more information.

### IPv6和双栈Services
IPv6 and dual stack services are supported in L2 mode, and in BGP mode only via the experimental FRR mode.

In order for MetalLB to allocate IPs to a dual stack service, there must be at least one address pool having both addresses of version v4 and v6.

Note that in case of dual stack services, it is not possible to use spec.loadBalancerIP as it does not allow to request for multiple IPs, so the annotation metallb.universe.tf/loadBalancerIPs must be used.

### IP地址共享
默认, Services 之间不能共享IP地址. 如果你希望多个Service使用一个IP地址. 你可以在 service 上配置 annotation `metallb.universe.tf/allow-shared-ip` 来开启优选择的IP地址共享.

这个 annotation 的值是一个共享 key. 下面几种情况下, Services 可以共享IP:

- 他们都有一个相同的共享 key.
- 他们使用的端口不一样(比如一个是 tcp/80 另一个是 tcp/443)
- 他们都使用`Cluster`外部流量策略, 或者它们都指向相同的一组 pods(比如 pod 选择器是完全相同的)

如果这些条件不满足, MetalLB可能会给两个 service 分配同一个IP, 但是也不一定. 如果你想确保多个 service 共享一个特定的IP, 使用上面提到的`spec.loadBalancerIP`来定义.

以这样的方式来管理 Service 有两个主要原因: 1. 规避 Kubernetes 的限制; 2. 可使用的IP地址有限.

下面是两个 services 共享一个IP地址的例子:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: dns-service-tcp
  namespace: default
  annotations:
    metallb.universe.tf/allow-shared-ip: "key-to-share-1.2.3.4"
spec:
  type: LoadBalancer
  loadBalancerIP: 1.2.3.4
  ports:
    - name: dnstcp
      protocol: TCP
      port: 53
      targetPort: 53
  selector:
    app: dns
---
apiVersion: v1
kind: Service
metadata:
  name: dns-service-udp
  namespace: default
  annotations:
    metallb.universe.tf/allow-shared-ip: "key-to-share-1.2.3.4"
spec:
  type: LoadBalancer
  loadBalancerIP: 1.2.3.4
  ports:
    - name: dnsudp
      protocol: UDP
      port: 53
      targetPort: 53
  selector:
    app: dns
```

Kubernetes does not currently allow multiprotocol LoadBalancer services. This would normally make it impossible to run services like DNS, because they have to listen on both TCP and UDP. To work around this limitation of Kubernetes with MetalLB, create two services (one for TCP, one for UDP), both with the same pod selector. Then, give them the same sharing key and `spec.loadBalancerIP` to colocate the TCP and UDP serving ports on the same IP address.

The second reason is much simpler: if you have more services than available IP addresses, and you can’t or don’t want to get more addresses, the only alternative is to colocate multiple services per IP address.

## 一些例子

As an example of how to use all of MetalLB’s options, consider an ecommerce site that runs a production environment and multiple developer sandboxes side by side. The production environment needs public IP addresses, but the sandboxes can use private IP space, routed to the developer offices through a VPN.

Additionally, because the production IPs end up hardcoded in various places (DNS, security scans for regulatory compliance…), we want specific services to have specific addresses in production. On the other hand, sandboxes come and go as developers bring up and tear down environments, so we don’t want to manage assignments by hand.

We can translate these requirements into MetalLB. First, we define two address pools, and set BGP attributes to control the visibility of each set of addresses:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production
  namespace: metallb-system
spec:
  # 生产使用,因为公网的IP很贵,我们只有4个
  addresses:
  - 42.176.25.64/30
```
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: sandbox
  namespace: metallb-system
spec:
  addresses:
  # 相反, 沙箱环境使用私有IP空间
  # 免费的而且很多, 所以我们给这个地址池分配了大量的IP
  # 这样开发者可以根据需要启动很多沙箱环境
  - 192.168.144.0/20
```

然后我们需要公布他们, 可以通过设置BGP的一些属性来控制每一个地址集合的可见性.

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: external
  namespace: metallb-system
spec:
  ipAddressPools:
  - production
```
```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: local
  namespace: metallb-system
spec:
  ipAddressPools:
  - sandbox
  communities:
    - vpn-only
```
```yaml
# 我们数据中心路由器知道“vpn-only”的BGP社区。
# 带有此社区标签的公告将仅通过公司VPN隧道传播回开发人员办公室。
apiVersion: metallb.io/v1beta1
kind: Community
metadata:
  name: communities
  namespace: metallb-system
spec:
  communities:
  - name: vpn-only
    value: 1234:1
```

在我们沙箱环境的 Helm 定义 charts 中, 我们给每一个 service 都打上了annotation `metallb.universe.tf/address-pool: sandbox`. 这样, 不管开发者什么时候创建沙箱环境, 都会从`192.168.144.0/20`获得一个IP地址.

对于生产环境, 我们使用`spec.loadBalancerIP`参数来给 service 定义特定的IP地址. 

---
```
Copyright © The MetalLB Contributors.
Copyright © 2021 The Linux Foundation ®. All rights reserved. Linux 基金会已注册商标并使用商标.
```