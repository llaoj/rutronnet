---
title: "[译]什么是 eBPF?"
description: "关于 eBPF 的入门介绍文章, 讲了其架构和原理等."
summary: "eBPF 程序是事件驱动的, 能在内核或应用程序执行到一个特定的 hook 点时执行. 预定义的 hooks 包含系统调用, 函数出/入口, 内核追踪点, 网络事件等等. 如果预定义 hook 不能满足需求, 也可以创建内核探针(kprobe)或者用户探针(uprobe), 在内核/用户应用程序的任何位置, 把探针附加到 eBPF 程序上."
date: "2022-03-23"
menu: "main"
tags:
- "bcc"
- "ebpf"
categories:
- "technology"
---

点击查看[原文](https://ebpf.io/what-is-ebpf)

## 什么是 eBPF?

eBPF 是革命性技术, 起源于 linux 内核, 能够在操作系统内核中执行沙盒程序. 旨在不改变内核源码或加载内核模块的前提下安全便捷的扩展内核能力.

历史上, 由于内核拥有全局查看并控制整个操作系统的特权, 操作系统一直被认为是实现可观察性, 安全, 网络功能的理想地方. 同时, 由于其核心角色和对于稳定和安全的高要求, 操作系统很难演进. 因此, 传统上与在操作系统之外实现的功能相比, 操作系统级别的创新率较低.

![overview](/posts/2203/what-is-ebpf/overview.png)

eBPF 从根本上改变了这种一成不变的状态. 通过允许在操作系统中执行沙盒程序, 开发者可以通过执行 eBPF 程序, 来给运行中的操作系统添加额外的能力. 就像在本地使用即时编译器(JIT)和验证引擎一样, 操作系统可以保证安全性和执行效率. 这催生了不少基于 eBPF 的项目, 涵盖了广泛的用例, 包括下一代网络、可观察性和安全功能.

今天, eBPF 被广泛用于各种用例: 在现代化的数据中心和云原生环境中提供高性能网络和负载均衡, 以较低的开销提取细粒度的可观察性安全数据, 帮助应用程序开发者追踪应用, 并能够在性能故障分析、预防性应用和容器运行时安全执法等方面提供帮助. 它的可能性是无限的, 关于 eBPF 的创新才刚开始.

### 什么是 eBPF.io?

eBPF.io 是以 eBPF 为主题, 每个人学习和协作的地方. eBPF 是一个开源社区, 每个人可以实践或者分享. 不论你是想阅读 eBPF 第一篇介绍文章, 还是发现更多阅读素材, 抑或是为变成 eBPF 主项目贡献者迈出第一步, eBPF.io 会一直陪伴你帮助你.

## 介绍 eBPF

下面的章节是关于 eBPF 的快速介绍. 如果你想了解更多, 查看 [eBPF & XDP Reference Guide](https://cilium.readthedocs.io/en/stable/bpf/). 不管你是一名从事 eBPF 的开发者, 或是有兴趣使用 eBPF 作为解决方案, 理解基础概念和架构都是很有用的.

### Hook 概览

eBPF 程序是事件驱动的, 能在内核或应用程序执行到一个特定的 hook 点时执行. 预定义的 hooks 包含系统调用, 函数出/入口, 内核追踪点, 网络事件等等.

![syscall_hook](/posts/2203/what-is-ebpf/syscall_hook.png)

如果预定义 hook 不能满足需求, 也可以创建内核探针(kprobe)或者用户探针(uprobe), 在内核/用户应用程序的任何位置, 把探针附加到 eBPF 程序上.

### eBPF 程序怎么写?

在很多场景中, 用户不需要直接使用 eBPF, 而是通过一些项目, 比如 [cilium](https://ebpf.io/projects/#cilium), [bcc](https://ebpf.io/projects/#bcc) 或 [bpftrace](https://ebpf.io/projects/#bpftrace), 它们是 eBPF 上层的抽象, 提供了使用 eBPF 实现的特定功能, 用户无需直接编写 eBPF 程序.

![clang](/posts/2203/what-is-ebpf/clang.png)

如果没有高级抽象, 就需要直接编写 eBPF 程序. Linux 内核要器加载字节码形式的 eBPF 程序. 虽然可以直接编写字节码, 但是更普遍的开发实践是借用像 [LLVM](https://llvm.org/) 这样的编译器, 把伪 C 代码编译成字节码.

### 加载器 & 验证架构

当所需的钩子被识别后, 可以使用 bpf 系统调用将 eBPF 程序加载到 Linux 内核中. 这通常使用一个可用的 eBPF 工具库来完成. 下一节将介绍一些可用的开发工具链.

![go](/posts/2203/what-is-ebpf/go.png)

当程序加载到 Linux 内核中时, 它在附加到请求的钩子之前要经过两个步骤:

### 验证

这一步是为了确保 eBPF 程序安全执行. 它验证程序是否满足一些条件, 比如:

- 加载 eBPF 程序的进程拥有所需的能力(特权). 除非启用非特权 eBPF, 否则只有特权进程才能加载 eBPF 程序.
- 该程序不能崩溃或者以其他方式伤害操作系统.
- 该程序必须总是能执行完(即程序不会死循环, 阻止后面的处理).

### 即时编译 (JIT)

该步骤将通用字节码翻译成机器特定的指令集, 以优化程序的执行速度. 这使 eBPF 程序像原生编译的内核代码或者像已加载的内核模块代码一样高效运行.

### Maps

eBPF 程序一个重要能力是: 能够共享收集的信息, 能够存储状态. 为了实现该能力, eBPF 程序借用 Maps 来存储/获取数据, 它支持丰富的数据结构. 通过系统调用, 可以从 eBPF 程序或者用户空间应用访问 maps.

![map_architecture](/posts/2203/what-is-ebpf/map_architecture.png)

为了解 map 类型的多样性, 下面是不完整的 map 类型列表. 这些类型的变量同时是 共享变量 和 per-CPU 变量.

- Hash tables, Arrays 哈希表, 数组
- LRU (Least Recently Used) 最近最少使用
- Ring Buffer 环形缓冲区
- Stack Trace 堆栈跟踪
- LPM (Longest Prefix Match) 最长前缀匹配
- ...

### 帮助函数

eBPF 程序不能随意调用内核函数. 如果允许的话, 将会把 eBPF 程序绑定到特定的内核版本, 这会使程序的兼容性复杂化. 所以, eBPF 程序转而使用帮助函数, 它是内核提供的大家熟知的稳定的 API.

![helper](/posts/2203/what-is-ebpf/helper.png)

可用的帮助函数还在持续发展中, 例如:

- 生成随机数
- 获取当前时间和日期
- 访问 eBPF map
- 获取 process/cgroup 上下文
- 网络数据包处理和转发逻辑

### 尾调用 & 函数调用

eBPF 程序可以组合使用尾调用和函数调用(tail & function calls). 函数调用允许在 eBPF 程序中定义和调用函数. 尾调用可以调用执行其他 eBPF 程序, 并替换执行上下文, 类似于 `execve()` 系统调用对常规进程的操作方式.

![tailcall](/posts/2203/what-is-ebpf/tailcall.png)

### eBPF 安全

_权利越大, 责任越大_

eBPF 是一项伟大的技术, 当下在很多关键软件中都扮演了核心的角色. 在 eBPF 程序开发过程中, 当 eBPF 进入 Linux 内核时, eBPF 的安全性就变得异常重要. eBPF 的安全性通过下面几点来保证:

#### 要求特权

除非开启非特权 eBPF, 所有企图加载 eBPF 程序到内核的进程必须在特权模式（root）下运行，或者必须获得 CAP_BPF 能力. 这意味着非授信的程序不能加载 eBPF 程序.

如果开启非特权 eBPF, 非特权进程可以加载特定的 eBPF 程序, 它们仅能使用被缩减的功能集合, 并且将受限制的访问内核.

#### 验证器

如果进程允许加载 eBPF 程序, 所有的程序都要经过 eBPF 验证器, 验证器来确保程序本身的安全性. 这意味着:

- 通过验证的程序一定会执行完, 比如, eBPF 程序不会卡住或死循环. eBPF 程序可以包含有边界的循环, 但是验证器要求, 循环必须具有可以被执行到的退出条件.
- 程序不能使用任何未初始化的变量或者越界访问内存.
- 程序必须在系统要求的大小范围内. 随意大的 eBPF 程序是无法加载的.
- 程序必须具备有限的复杂性. 验证器会评估所有可能的执行路径, 并且必须在配置的复杂度范围内完成分析.

#### 加固

完成验证之后, 根据 eBPF 程序是从特权进程还是非特权进程加载, 来决定是否加固的 eBPF 程序. 这包括:

- **程序执行保护**: 存有 eBPF 程序的内核内存是被保护的并且是只读的. 不管是内核 bug 或者是被恶意操纵, 内核都将崩溃, 而不是允许它继续执行损坏/被操纵的程序.
- **Mitigation against Spectre**: Under speculation CPUs may mispredict branches and leave observable side effects that could be extracted through a side channel. 举几个例子: eBPF programs mask memory access in order to redirect access under transient instructions to controlled areas, the verifier also follows program paths accessible only under speculative execution and the JIT compiler emits Retpolines in case tail calls cannot be converted to direct calls.
- **常量 blinding**: 代码中的所有常量都被 blinded, 以防止 JIT spraying 攻击. 这可以避免: 当存在某种内核 bug 的情况下, 攻击者可以把可执行代码作为常量注入, 从而让攻击者跳转到 eBPF 程序的内存区域来执行代码.

#### 抽象的运行时上下文

eBPF 程序不能直接访问任意内核内存. 必须通过 **eBPF 助手函数**访问位于程序上下文之外的数据和数据结构. 这保证了一致性的数据访问, 并使任何此类访问均受制于 eBPF 程序的权限, 例如如果可以保证修改是安全的, 则允许运行的 eBPF 程序修改某些数据结构的数据. eBPF 程序不能随机修改内核中的数据结构.

## 为什么使用 eBPF?

### 可编程的力量

还记得 GeoCities 吗? 20年前, 网页几乎全都是用静态标记语言(HTML)写的, 网页基本上是一种应用程序(浏览器)能打开的文件. 再看今天, 网页已经变成了非常成熟的应用, 并且 WEB 已经取代了绝大部分编译语言写的应用. 是什么成就了这次革命?

![geocities](/posts/2203/what-is-ebpf/geocities.png)

简单来说, 就是引入 JavaScript 之后的可编程性. 它开启了一场大规模的革命, 几乎将浏览器变成了独立的操作系统.

为什么呢? 程序员不再受限于特定的浏览器版本. 没有去说服标准机构去定义更多需要的 HTML 标签, 相反, 而是提供了一些必要的构建模块, 将浏览器底层的演进和运行在其上层的应用进行分离. 这样说可能过于简单, 因为 HTML 的确做了不小的贡献, 也的确有所发展, 但是 HTML 本身的变革还不够.

在举这个例子并将其应用到 eBPF 之前, 让我们看一下对引入 JavaScript 至关重要的几个关键方面:

- **安全性**: 不受信任的代码在用户的浏览器中运行. 这是通过沙盒 JavaScript 程序和抽象对浏览器数据的访问来解决的.
- **持续交付**: 在不需要浏览器发新版本的情况下, 程序要能不断更新. 这得益于浏览器低级的(low-level)构建模块, 它能构建任意的逻辑.
- **性能**: 必须以最小的开销提供可编程性. 这得益于即时编译器(JIT).

上面说的所有内容, 在 eBPF 中都能找到:

### eBPF 对 Linux 内核的影响

现在我们回到 eBPF. 为了理解 eBPF 可编程性在 Linux 内核上的影响, 我们来看张图片, 它有助于我们对 Linux 内核的架构进行理解, 并且能了解它是如何与应用程序和硬件进行交互的.

![kernel_arch](/posts/2203/what-is-ebpf/kernel_arch.png)

Linux 内核的主要目的是抽象硬件或虚拟硬件, 并提供一致的 API(系统调用), 允许应用程序运行和共享资源. 为了实现这一点, 维护了大量的子系统和层来分配这些职责. 每个子系统通常允许某种级别的配置来满足不同的用户需求. 如果没办法通过配置满足某种需求, 则需要更改内核. 从历史上看, 有两种选择:
 
|原生支持	|内核模块 |
|:-|:-|
|1. 更改内核源代码并说服 Linux 内核社区 |1. 写一个新的内核模块 |
|2. 等几年新内核版本上市 |2. 定期修复它, 因为每个内核版本都可能破坏它 |
| |3. 由于缺乏安全边界, 有损坏 Linux 内核的风险 |

在不需要改变内核源码或者加载内核模块的情况下, eBPF 为重新编程内核行为提供了一种新的选择. 在很多地方, 这很像 JavaScript 和其他脚本语言, 它们让那些改变难度大, 成本高的系统开始演进.

## 开发工具链

有几个开发工具链来能够协助 eBPF 程序的开发和管理. 它们能满足用户的不同需求:

### bcc

BCC 是一个框架, 能够让用户编写嵌入了 eBPF 程序的 python 程序. 该框架主要用来分析和跟踪应用/系统, eBPF 在其中主要负责收集统计数据或生成事件, 然后, 对应的用户空间程序会收集这些数据并以易读的方式进行展示. 运行 python 程序会生成 eBPF 字节码并将其加载进内核.

![bcc](/posts/2203/what-is-ebpf/bcc.png)

### bpftrace

bpftrace 是 Linux eBPF 的高级跟踪语言, 可用于最新的 Linux 内核(4.x). bpftrace 使用 LLVM 作为后端将脚本编译为 eBPF 字节码，并利用 BCC 与 Linux eBPF 子系统以及现有的 Linux 跟踪功能进行交互: 内核动态跟踪(kprobes)、用户级动态跟踪(uprobes)和跟踪点(tracepoints). bpftrace 语言的灵感来自 awk、C 和以前的跟踪器(如 DTrace 和 SystemTap).

![bpftrace](/posts/2203/what-is-ebpf/bpftrace.png)

### eBPF Go 类库

eBPF Go 库提供了一个通用的 eBPF 库, 它将获取 eBPF 字节码的过程与 eBPF 程序的加载和管理分离. eBPF 程序通常是通过编写高级语言创建的, 然后使用 clang/LLVM 编译器编译为 eBPF 字节码.

![go](/posts/2203/what-is-ebpf/go.png)

### libbpf C/C++ 类库

libbpf 库是一个基于 C/C++ 的通用 eBPF 库. 它提供给应用程序一种易用的 API 来抽象化 BPF 系统调用, 并将 eBPF 字节码(clang/LLVM 编译器生成)加载到内核的过程与之分离.

![libbpf](/posts/2203/what-is-ebpf/libbpf.png)

## 阅读更多

如果你想学习更多的 eBPF 知识, 阅读下面的材料:

### 文档

- [BPF & XDP Reference Guide](https://docs.cilium.io/en/stable/bpf/)  
  Cilium 文档, 2020年8月
- [BPF Documentation](https://www.kernel.org/doc/html/latest/bpf/index.html)  
  Linux 内核中的 BPF 介绍文档
- [BPF Design Q&A](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/bpf/bpf_design_QA.rst)  
  内核相关的 eBPF 问答

### 教程

- [Learn eBPF Tracing: Tutorial and Examples](https://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html)  
  Brendan Gregg 的博客, 2019年1月
- [XDP Hands-On Tutorials](https://github.com/xdp-project/xdp-tutorial)  
  很多作者, 2019年
- [BCC, libbpf and BPF CO-RE Tutorials](https://facebookmicrosites.github.io/bpf/blog/)  
  Facebook 的 BPF 博客, 2020年

### 发言

#### 基础

- [eBPF and Kubernetes: Little Helper Minions for Scaling Microservices (Slides)]()  
  Daniel Borkmann, KubeCon EU, Aug 2020
- [eBPF - Rethinking the Linux Kernel (Slides)]()  
  Thomas Graf, QCon London, April 2020
- [BPF as a revolutionary technology for the container landscape (Slides)]()  
  Daniel Borkmann, FOSDEM, Feb 2020
- [BPF at Facebook]()  
  Alexei Starovoitov, Performance Summit, Dec 2019
- [BPF: A New Type of Software (Slides)]()  
  Brendan Gregg, Ubuntu Masters, Oct 2019
- [The ubiquity but also the necessity of eBPF as a technology]()  
  David S. Miller, Kernel Recipes, Oct 2019

#### 深入

- [BPF and Spectre: Mitigating transient execution attacks (Slides)]()  
  Daniel Borkmann, eBPF Summit, Aug 2021
- [BPF Internals (Slides)]()  
  Brendan Gregg, USENIX LISA, Jun 2021

#### Cilium

- [Advanced BPF Kernel Features for the Container Age (Slides)]()  
  Daniel Borkmann, FOSDEM, Feb 2021
- [Kubernetes Service Load-Balancing at Scale with BPF & XDP (Slides)]()  
  Daniel Borkmann & Martynas Pumputis, Linux Plumbers, Aug 2020
- [Liberating Kubernetes from kube-proxy and iptables (Slides)]()  
  Martynas Pumputis, KubeCon US 2019
- [Understanding and Troubleshooting the eBPF Datapath in Cilium (Slides)]()  
  Nathan Sweet, KubeCon US 2019
- [Transparent Chaos Testing with Envoy, Cilium and BPF (Slides)]()  
  Thomas Graf, KubeCon EU 2019
- [Cilium - Bringing the BPF Revolution to Kubernetes Networking and Security (Slides)]()  
  Thomas Graf, All Systems Go!, Berlin, Sep 2018
- [How to Make Linux Microservice-Aware with eBPF (Slides)]()  
  Thomas Graf, QCon San Francisco, 2018
- [Accelerating Envoy with the Linux Kernel]()  
  Thomas Graf, KubeCon EU 2018
- [Cilium - Network and Application Security with BPF and XDP (Slides)]()  
  Thomas Graf, DockerCon Austin, Apr 2017

#### Hubble

- [Hubble - eBPF Based Observability for Kubernetes]()  
  Sebastian Wicki, KubeCon EU, Aug 2020

### 图书

- [Systems Performance: Enterprise and the Cloud, 2nd Edition]()  
  Brendan Gregg, Addison-Wesley Professional Computing Series, 2020
- [BPF Performance Tools]()  
  Brendan Gregg, Addison-Wesley Professional Computing Series, Dec 2019
- [Linux Observability with BPF]()  
  David Calavera, Lorenzo Fontana, O'Reilly, Nov 2019

### 文章 & 博客

- [BPF for security - and chaos - in Kubernetes]()  
  Sean Kerner, LWN, Jun 2019
- [Linux Technology for the New Year: eBPF]()  
  Joab Jackson, Dec 2018
- [A thorough introduction to eBPF](https://lwn.net/Articles/740157/)  
  Matt Fleming, LWN, Dec 2017
- [Cilium, BPF and XDP]()  
  Google Open Source Blog, Nov 2016
- [Archive of various articles on BPF]()  
  LWN, since Apr 2011
- [Various articles on BPF by Cloudflare]()  
  Cloudflare, since March 2018
- [Various articles on BPF by Facebook]()  
  Facebook, since August 2018
