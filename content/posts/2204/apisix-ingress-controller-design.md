---
title: "Apisxi Ingress Controller 设计说明"
description: "Apisxi Ingress Controller 设计说明"
summary: "Grafana Mimir 是目前最具扩展性、性能最好的开源时序数据库，Mimir 允许你将指标扩展到 1 亿。它部署简单、高可用、多租户支持、持久存储、查询性能超高，比 Cortex 快 40 倍。 Mimir 托管在 https://github.com/grafana/mimir 并在 AGPLv3 下获得许可。"
date: "2022-04-14"
menu: "main"
tags:
- apisix
- kubernetes
categories:
- "技术"
---

## 架构设计

Apache APISIX - 专门为 kubernetes 研发的入口控制器。

apisix-ingress-controller 需要的所有配置都是通过 Kubernetes CRDs (Custom Resource Definitions) 定义的。支持在 Apache APISIX 中配置插件、上游的服务注册发现机制、负载均衡等。  
apisix-ingress-controller 是 Apache APISIX 的控制面组件. 当前服务于 Kubernetes 集群. 未来, 计划分离出子模块以适配更多的部署模式，比如虚拟机集群部署。架构图如下  

![architecture](/posts/2204/apisix-ingress-controller-design/arch.png)

## 状态

该项目目前是 general availability 级别

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

## 内部架构

![internal-arch](/posts/2204/apisix-ingress-controller-design/internal-arch.png)

## 先决条件

apisix-ingress-controller 要求 kubernetes 版本 1.16+. 因为使用了 CustomResourceDefinition v1 stable 版本的 API. 
从 1.0.0 版本开始，APISIX-ingress-controller 要求 Apache APISIX 版本 2.7+.

## 时序图

apisix-ingress-controller 负责和 Kubernetes Apiserver 交互, 申请可访问资源权限（RBAC），监控变化，在 Ingress 控制器中实现对象转换，比较变化，然后同步到 Apache APISIX。

![flow](/posts/2204/apisix-ingress-controller-design/flow.png)

这是一张流程图，介绍了ApisixRoute和其他CRD在同步过程中的主要逻辑

![sync-logic-controller](/posts/2204/apisix-ingress-controller-design/sync-logic-controller.png)

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

## 用户故事
[思必驰：为什么我们重新写了一个 k8s ingress controller？](https://mp.weixin.qq.com/s/bmm2ibk2V7-XYneLo9XAPQ)  
[腾讯云：为什么选择 apisix 实现 kubernetes ingress controller](https://www.upyun.com/opentalk/448.html)
