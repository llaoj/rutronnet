---
title: "kubernetes 中的 pod 究竟是什么"
description: "kubernetes 中 pod 的设计是一个伟大的发明, 今天我很有必要去聊一下 pod 和 container, 探究一下它们究竟是什么? kubernetes 官方文档中关于 pod 概念介绍提供了一个完整的解释, 但写的不够详细, 表达过于专业, 但还是很推荐大家阅读一下. 当然这篇文档应该更接地气."
date: "2022-02-08"
menu: "main"
tags:
- "pod"
- "kubernetes"
categories:
- "technology"
---

## 前言

kubernetes 中 pod 的设计是一个伟大的发明, 今天我很有必要去聊一下 pod 和 container, 探究一下它们究竟是什么? kubernetes 官方文档中关于[pod 概念介绍](https://kubernetes.io/zh/docs/concepts/workloads/pods/#pod-storage)提供了一个完整的解释, 但写的不够详细, 表达过于专业, 但还是很推荐大家阅读一下. 当然这篇文档应该更接地气.

## 容器真的存在吗?

linux 中是没有容器这个概念的, 容器就是 linux 中的普通进程, 它使用了 linux 内核提供的两个重要的特性: namespace & cgroups. 

namespace 提供了一种隔离的特性, 让它之外的内容隐藏, 给它下面的进程一个不被干扰的运行环境(其实不完全,下面说) .

namespace 包含:

- hostname
- Process IDs
- File System
- Network Interface
- Inter-Process Communication(IPC)

接上面, 其实 namespace 内部的进程并不是完全不和外面的进程产生影响的. 进程可以不受限制的使用物理机上的所有资源, 这样就会导致其他进程无资源可用. 所以, 为了限制进程资源使用, linux 提供了另一种特性 cgroups.  进程可以像在 namespace 中运行, 但是 cgroups 限制了进程的可以使用的资源. 这些资源包括:

- CPU
- RAM
- block I/O
- network I/O
- etc.

CPU 通常按照毫核来限制(单位:m), 1000m=1C;  内存按照RAM的字节数来限制. 进程可以在 cgroups 设置的资源限制范围内运行, 不允许超限使用, 比如, 超过内存限制就会报 OOM(out of memory) 的错误.

需要特别说明的是, 上面提到的 namespace & cgroup 都是 Linux 独立的特性, 你可以使用上面提到的 namespace 中的一个或者多个. namespace & cgroup 作用到一组或者一个进程上. 你可以把多个进程放在一个 namespace 中, 这样它们就可以彼此交互, 或者 把他们放在一个 cgroups 中, 这样他们就可以共享一个CPU & Mem 资源限制.

## 组合容器

我们都用过 docker, 当我们启动一个容器的时候, docker 会帮我们给每一个容器创建它们自己的 namespace & cgroups. 这应该就是我们理解的容器.

![image-20210809102424781](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809102430362.png)

如图, 容器本身还是比较独立的, 他们可能会有映射到主机的端口和卷, 这样就可以和外面通信. 但是我们也可以通过一些命令将多个容器组合到一组namespace中, 下面我们举个例子说明:

首先, 创建一个 nginx 容器:

```sh
# cat <<EOF >> nginx.conf
> error_log stderr;
> events { worker_connections  1024; }
> http {
>     access_log /dev/stdout combined;
>     server {
>         listen 80 default_server;
>         server_name example.com www.example.com;
>         location / {
>             proxy_pass http://127.0.0.1:2368;
>         }
>     }
> }
> EOF
# docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf -p 8080:80 --ipc=shareable nginx 
```

接着, 我们再启动一个 ghost 容器, ghost 是一个开源的博客系统, 同时我们添加几个额外的命令到 nginx 容器上.

```
# docker run -d --name ghost --net=container:nginx --ipc=container:nginx --pid=container:nginx ghost
```

好了, 现在 nginx 容器可以通过 localhost 将请求代理到 ghost 容器, 访问 `http://localhost:8080`试试, 你可以通过 nginx 反向代理看到一个 ghost 博客. 上面的命令就把一组容器组合到里同一组 namespace 中, 容器彼此之间可以互相发现/通信. 

就像这样:

![image-20210809134007587](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809134007587.png)

## 某种意义上, pod 就是一组容器

现在我们已经知道, 我们可以把一组进程组合到一个 namespace & cgroups 中, 这就是 kubernetes 中的 pod.  pod 允许你定义你要运行的容器, 然后 kubernetes 会帮正确的配置 namespace & cgroups. 它稍微复杂的一点是, 网络这块它没用 docker network, 而是用到了 CNI(通用网络接口), 但原理都差不多.

![image-20210809135653975](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809135653975.png)

按照上述方式创建的 pod, 更像是运行在同一台机器上, 他们之间可以通过 localhost 通信, 可以共享存储卷. 甚至他们可以使用 IPC 或者互相发送 HUP / TERM 这类信号.

我们再举个例子, 如下图, 我们运行一个 nginx 反向代理 app,  再运行一个 confd, 当 app 实例增加或减少的时候去动态配置 `nginx.conf` 并重启 nginx, etcd 中存储了 app 的 ip 地址. 当 ip 列表发生变化, confd 会收到 etcd 发的通知, 并更新 `nginx.conf` 并给 nginx 发送一个 HUP 信号, nginx 收到 HUP 信号会重启.

![image-20210809141738720](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809141738720.png)

如果用 docker, 你大概会把 nginx 和 confd 放在一个容器中. 由于 docker 只有一个 entrypoint, 所以你要启动一个类似 supervisord 一样的进程管理器 来让 nginx 和 confd 都运行起来. 你每启动一个 nginx 副本就要启动一个 supervisord, 这不好吧. 更重要的是, docker 只知道 supervisord 的状态, 因为它只有一个 entrypoint. 它看不到里面的所有进程, 这就意味着, 你用 docker 提供的工具获取不到他们的信息. 一旦 nginx `Crash-Restart Loop`, docker 一点办法没有.

![image-20210809142718304](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809142718304.png)

通过 pod , kubernetes 能管理每一个进程, 看到他们的状态, 它可以通过 api 将进程状态信息暴露给用户, 或者提供进程崩溃时重启/记录日志等服务.

![image-20210809143225671](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809143225671.png)

## 把容器当作接口

使用 pod 这种组织容器的方式, 可以把容器当作提供各种功能的 "接口". 它不同于传统意义上的 web 接口. 更像是可以被容器所使用的某种抽象意义的接口.

我们拿上面 nginx+confd 的例子来说, confd 不需要知道任何 nginx 进程的东西, 它就只需要去 watch etcd 然后给 nginx 进程发送 HUP 信号或者执行个命令. 而且你可以把 nginx 替换成其他任何类型的应用, 以这样的模式来使用 confd 的这种能力. 这种模式下, confd 通常被称作 **"sidecar container"**  边车容器, 下面这图就很形象.

![image-20210809150256978](/posts/2202/what-are-kubernetes-pods-anyway.assets/image-20210809150256978.png)

像 istio 这样的服务网格项目, 也是, 给应用程序容器放置一个边车容器来提供服务路由, 遥测, 网络策略等功能, 但是对应用程序并没有做任何侵略性更改. 你也可以使用多个边车容器来组织 pod, 比如在一个 pod 中同时放置 confd & istio 边车容器. 用这样的方式, 可以构建更加复杂可靠的系统, 同时还能保持每个应用的独立性和简单性.

## 参考

[What are Kubernetes Pods Anyway?](https://www.ianlewis.org/en/what-are-kubernetes-pods-anyway)

[What even is a container: namespaces and cgroups](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)

video: [Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8)