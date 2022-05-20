---
title: "Apisxi Ingress Controller 设计说明"
description: "Apisxi Ingress Controller 设计说明"
summary: "apisix-ingress-controller 要求 kubernetes 版本 1.16+. 因为使用了 CustomResourceDefinition v1 stable 版本的 API.
从 1.0.0 版本开始，APISIX-ingress-controller 要求 Apache APISIX 版本 2.7+."
date: "2022-04-14"
menu: "main"
tags:
- apisix
- kubernetes
categories:
- "technology"
---

Apache APISIX - 专门为 kubernetes 研发的入口控制器。

## 状态

该项目目前是 general availability 级别

## 先决条件

apisix-ingress-controller 要求 kubernetes 版本 1.16+. 因为使用了 CustomResourceDefinition v1 stable 版本的 API.
从 1.0.0 版本开始，APISIX-ingress-controller 要求 Apache APISIX 版本 2.7+.

## 功能特性

- 使用 Custom Resource Definitions(CRDs) 对 Apache APISIX 进行声明式配置，使用 kubernetes yaml 结构最小化学习成本。
- Yaml 配置热加载
- 支持原生 Kubernetes Ingress (v1 和 v1beta1) 资源
- Kubernetes endpoint 自动注册到 Apache APISIX 上游节点
- 支持基于 POD（上游节点） 的负载均衡
- 开箱支持上游节点健康检查
- 扩展插件支持热配置并且立即生效
- 支持路由的 SSL 和 mTLS
- 支持流量切割和金丝雀发布
- 支持 TCP 4层代理
- Ingress 控制器本身也是一个可插拔的热加载组件
- 多集群配置分发

[这里有一份在线竞品分析表格](https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k/edit#gid=907731238)

### Apache APISIX Ingress vs. Kubernetes Nginx Ingress

- yaml 配置热加载
- 更方便的金丝雀发布
- 配置验证，安全可靠
- 丰富的插件和生态, [插件列表](https://github.com/apache/apisix/tree/master/docs/en/latest/plugins)
- 支持 APISIX 自定义资源和原生 kubernetes ingress 资源
- 更活跃的社区

## 设计原理

### 架构

apisix-ingress-controller 需要的所有配置都是通过 Kubernetes CRDs (Custom Resource Definitions) 定义的。支持在 Apache APISIX 中配置插件、上游的服务注册发现机制、负载均衡等。  
apisix-ingress-controller 是 Apache APISIX 的控制面组件. 当前服务于 Kubernetes 集群。 未来, 计划分离出子模块以适配更多的部署模式，比如虚拟机集群部署。

整体架构图如下：

![architecture](/posts/2204/apisix-ingress-controller-design/arch.png)

这是一张内部架构图：

![internal-arch](/posts/2204/apisix-ingress-controller-design/internal-arch.png)

### 时序/流程图

apisix-ingress-controller 负责和 Kubernetes Apiserver 交互, 申请可访问资源权限（RBAC），监控变化，在 Ingress 控制器中实现对象转换，比较变化，然后同步到 Apache APISIX。

![flow](/posts/2204/apisix-ingress-controller-design/flow.png)

这是一张流程图，介绍了ApisixRoute和其他CRD在同步过程中的主要逻辑

![sync-logic-controller](/posts/2204/apisix-ingress-controller-design/sync-logic-controller.png)

### 结构转换

apisix-ingress-controller 给 CRDs 提供了外部配置方法。它旨在务于需要日常操作和维护的运维人员，他们需要经常处理大量路由配置，希望在一个配置文件中处理所有相关的服务，同时还希望能具备便捷和易于理解的管理能力。但是，Apache APISIX 则是从网关的角度设计的，并且所有的路由都是独立的。这就导致了两者在数据结构上存在差异。一个注重批量定义，一个注重离散实现。  
考虑到不同人群的使用习惯，CRDs 的数据结构借鉴了 Kubernetes Ingress 的数据结构，数据结构基本一致。
关于这两者的差别，请看下面这张图：

![struct-compare](/posts/2204/apisix-ingress-controller-design/struct-compare.png)

可以看到，它们是多对多的关系。因此，apisix-ingress-controller 必须对 CRD 做一些转换，以适应不同的网关。

### 规则比较

seven 模块内部保存了内存数据结构，目前与Apache APISIX资源对象非常相似。当 Kubernetes 资源对象有新变化时，seven 会比较内存对象，并根据比较结果进行增量更新。  
目前的比较规则是根据route/service/upstream资源对象的分组，分别进行比较，发现差异后做出相应的广播通知。

![diff-rules](/posts/2204/apisix-ingress-controller-design/diff-rules.png)

### 服务发现

根据 `ApisixUpstream` 中定义的 `namespace` `name` `port` 字段，apisix-ingress-controller 会在 Apache APISIX Upstream 中注册 处于 running 状态的 endpoints 节点，并且根据 kubernetes endpoints 状态进行实时同步。  
基于服务发现，apisix-ingress-controller 可以直接访问后端 pod，绕过 Kubernetes Service，可以实现自定义的负载均衡策略。

### Annotation 实现

不像 Kubernetes Nginx Ingress Controller，apisix-ingress-controller 的 annotation 实现是基于 Apache APISIX 的插件机制的。  
比如，可以通过在`ApisixRoute`资源对象中设置`k8s.apisix.apache.org/whitelist-source-range`annotation来配置白名单。

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  annotations:
    k8s.apisix.apache.org/whitelist-source-range: 1.2.3.4,2.2.0.0/16
  name: httpserver-route
spec:
    ...
```

黑/白名单功能是通过[ip-restriction](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/ip-restriction.md)插件来实现的。  
为方便的定义一些常用的配置，未来会有更多的 annotation 实现，比如CORS。

## ApisixRoute 介绍

ApisixRoute 是一个 CRD 资源，它关注如何将流量发送到后端，它有很多 APISIX 支持的特性。相比 Ingress，功能实现的更原生，语意更强。

### 基于路径的路由规则

URI 路径总是用于拆分流量，比如访问 foo.com 的请求， 含有 /foo 前缀请求路由到 foo 服务，访问 /bar 的请求要路由到 bar 服务。以 ApisixRoute 方式配置应该是这样的：

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: foo-bar-route
spec:
  http:
  - name: foo
    match:
      hosts:
      - foo.com
      paths:
      - "/foo*"
    backends:
     - serviceName: foo
       servicePort: 80
  - name: bar
    match:
      paths:
        - "/bar"
    backends:
      - serviceName: bar
        servicePort: 80
```

有`prefix`和`exact`两种路径类型可用 默认`exact`，当需要前缀匹配的时候，就在路径后加 * 比如 /id/* 能匹配所有带 /id/ 前缀的请求。

### 高级路由特性

基于路径的路由是最普遍的，但这并不够， 再试一下其他路由方式，比如 `methods` 和 `exprs`

`methods` 通过 HTTP 动作来切分流量，下面例子会把所有 GET 请求路由到 foo 服务（kubernetes service）

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: method-route
spec:
  http:
    - name: method
      match:
        paths:
          - /
        methods:
          - GET
      backends:
        - serviceName: foo
          servicePort: 80
```

`exprs`允许用户使用 HTTP 中的任意字符串来配置匹配条件，例如query、HTTP Header、Cookie。它可以配置多个表达式，而这些表达式又由主题(subject)、运算符(operator)和值/集合(value/set)组成。比如：

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: method-route
spec:
  http:
    - name: method
      match:
        paths:
          - /
        exprs:
          - subject:
              scope: Query
              name: id
            op: Equal
            value: "2143"
      backends:
        - serviceName: foo
          servicePort: 80
```

上面是绝对匹配，匹配所有请求的 query 字符串中 id 的值必须等于 2143。

#### 服务解析粒度

默认，apisix-ingress-controller 会监听 service 的引用，所以最新的 endpoints 列表会被更新到 Apache APISIX。同样 apisix-ingress-controller 也可以直接使用 service 自身的 clusterIP。如果这正是你想要的，配置 `resolveGranularity: service`(默认`endpoint`). 如下：

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: method-route
spec:
  http:
    - name: method
      match:
        paths:
          - /*
        methods:
          - GET
      backends:
        - serviceName: foo
          servicePort: 80
          resolveGranularity: service
```

### 基于权重的流量切分

这是 APISIX Ingress Controller 一个非常棒的特性。一个路由规则中可以指定多个后端，当多个后端共存时，将应用基于权重的流量拆分（实际上是使用Apache APISIX中的流量拆分 [traffic-split](https://apisix.apache.org/zh/docs/apisix/plugins/traffic-split/) 插件）您可以为每个后端指定权重，默认权重为 100。比如：

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: method-route
spec:
  http:
    - name: method
      match:
        paths:
          - /*
        methods:
          - GET
        exprs:
          - subject:
              scope: Header
              name: User-Agent
            op: RegexMatch
            value: ".*Chrome.*"
      backends:
        - serviceName: foo
          servicePort: 80
          weight: 100
        - serviceName: bar
          servicePort: 81
          weight: 50
```
上面有一个路由规则（1.所有`GET /*`请求 2.Header中有匹配 `User-Agent: .*Chrome.*` 的条目）它有两个后端服务 foo、bar，权重是100：50，意味着有2/3的流量会进入 foo，有1/3的流量会进入bar。

### 插件

Apache APISIX 提供了 40 多个插件，可以在 APIsixRoute 中使用。所有配置项的名称与 APISIX 中的相同。

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: httpbin-route
spec:
  http:
    - name: httpbin
      match:
        hosts:
        - local.httpbin.org
        paths:
          - /*
      backends:
        - serviceName: foo
          servicePort: 80
      plugins:
        - name: cors
          enable: true
```

为到 local.httpbin.org 的请求都配置了 Cors 插件

### Websocket 代理

创建一个 route，配置特定的 websocket 字段，就可以代理 websocket 服务。比如：

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: ws-route
spec:
  http:
    - name: websocket
      match:
        hosts:
          - ws.foo.org
        paths:
          - /*
      backends:
        - serviceName: websocket-server
          servicePort: 8080
      websocket: true
```

### TCP 路由

apisix-ingress-controller 支持基于端口的 tcp 路由

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: tcp-route
spec:
  stream:
    - name: tcp-route-rule1
      protocol: TCP
      match:
        ingressPort: 9100
      backend:
        serviceName: tcp-server
        servicePort: 8080
```
进入 apisix-ingress-controller 9100 端口的 TCP 流量会路由到后端 tcp-server 服务。  

**注意：** APISIX不支持动态监听，所以需要在APISIX[配置文件](https://github.com/apache/apisix/blob/master/conf/config-default.yaml#L111)中预先定义9100端口。

### UDP 路由

apisix-ingress-controller 支持基于端口的 udp 路由

```yaml
apiVersion: apisix.apache.org/v2beta3
kind: ApisixRoute
metadata:
  name: udp-route
spec:
  stream:
    - name: udp-route-rule1
      protocol: UDP
      match:
        ingressPort: 9200
      backend:
        serviceName: udp-server
        servicePort: 53
```

进入 apisix-ingress-controller 9200 端口的 TCP 流量会路由到后端 udp-server 服务。

**注意：** APISIX不支持动态监听，所以需要在APISIX[配置文件](https://github.com/apache/apisix/blob/master/conf/config-default.yaml#L111)中预先定义9200端口。

## ApisixUpstream 介绍

ApisixUpstream 是 kubernetes service 的装饰器。它设计成与其关联的 kubernetes service 的名字一致，将其变得更加强大，使该 kubernetes service 能够配置负载均衡策略、健康检查、重试、超时参数等。  
通过 ApisixUpstream 和 kubernetes service，apisix-ingress-controller 会生成 APISIX Upstream(s).  

### 配置负载均衡

需要适当的负载均衡算法来合理地分散 Kubernetes Service 的请求

```yaml
apiVersion: apisix.apache.org/v1
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  loadbalancer:
    type: ewma
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  selector:
    app: httpbin
  ports:
  - name: http
    port: 80
    targetPort: 8080
```
上面这个例子给 httpbin 服务配置了 ewma 负载均衡算法。有时候可能会需要会话保持，你可以配置一致性哈希负载均衡算法

```yaml
apiVersion: apisix.apache.org/v1
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  loadbalancer:
    type: chash
    hashOn: header
    key: "user-agent"
```

这样 apisix 就会根据 user-agent header 来分发流量。

### 配置健康检查

尽管 kubelet 已经提供了检测 pod 健康的探针机制。你可能还需要更加丰富的健康检查机制，比如被动健康检查机制。

```yaml
apiVersion: apisix.apache.org/v1
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  healthCheck:
    passive:
      unhealthy:
        httpCodes:
          - 500
          - 502
          - 503
          - 504
        httpFailures: 3
        timeout: 5s
    active:
      type: http
      httpPath: /healthz
      timeout: 5s
      host: www.foo.com
      healthy:
        successes: 3
        interval: 2s
        httpCodes:
          - 200
          - 206
```

上面的yaml片段定义了被动健康检查器来检查endpoints的健康状况。一旦连续三次请求的响应状态码是错误（500 502 503 504 中的一个），这个endpoint就会被标记成不健康并不会再给它分配流量了，直到它再次健康。  
所以，主动健康检查器就出现了。endpoint 可能掉线一段时间又回复健康，主动健康检查器主动探测这些不健康的endpoints，一旦满足健康条件就将其恢复为健康（条件：连续三次请求响应状态码为200或206）  
> 注意：主动健康检查器在某种程度上与 liveness/readiness 探针重复，但如果使用被动健康检查机制，则它是必需的。因此，一旦您使用了 ApisixUpstream 中的健康检查功能，主动健康检查器是强制性的。

### 配置重试和超时

当请求出现错误，比如网络问题或者服务不可用当时候，你可能想重试请求。默认重试次数是1，通过定义`retries`字段可以改变这个值。  
下面这个例子将`retries`定义为3，表明会对kubernetes service/httpbin的endpoints最多请求3次。  
> 注意：只有在尚未向客户端响应任何内容的情况下，才有可能将请求重试传递到下一个端点。也就是说，如果在传输响应的过程中发生错误或超时，就不会重试了。

```yaml
apiVersion: apisix.apache.org/v1
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  retries: 3
```

默认，connect、send 和 read 的超时时间是60s，这可能对有些应用不合适，修改`timeout`字段来改变默认值。

```yaml
apiVersion: apisix.apache.org/v1
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  timeout:
    connect: 5s
    read: 10s
    send: 10s
```
上面例子将connect、read 和 send 分别设置为 5s、10s、10s。

### 端口级别配置

有时，单个 kubernetes service 可能会暴露多个端口，这些端口提供不同的功能并且需要不同的上游配置。在这种情况下，您可以为单个端口创建配置。

```yaml
apiVersion: apisix.apache.org/v1
kind: ApisixUpstream
metadata:
  name: foo
spec:
  loadbalancer:
    type: roundrobin
  portLevelSettings:
  - port: 7000
    scheme: http
  - port: 7001
    scheme: grpc
---
apiVersion: v1
kind: Service
metadata:
  name: foo
spec:
  selector:
    app: foo
  portLevelSettings:
  - name: http
    port: 7000
    targetPort: 7000
  - name: grpc
    port: 7001
    targetPort: 7001
```

foo 服务暴露的两个端口，一个使用http协议，另一个使用grpc协议。同时，ApisixUpstream/foo 为7000端口配置http协议，为7001端口配置grpc协议（所有端口都是service端口），两个端口都共享使用同一个负载均衡算法。  
如果服务仅公开一个端口，则 PortLevelSettings 不是必需的，但在定义多个端口时很有用。

## 用户故事
[思必驰：为什么我们重新写了一个 k8s ingress controller？](https://mp.weixin.qq.com/s/bmm2ibk2V7-XYneLo9XAPQ)  
[腾讯云：为什么选择 apisix 实现 kubernetes ingress controller](https://www.upyun.com/opentalk/448.html)
