---
title: "Prometheus Operator 设计思路"
description: "这篇文章主要介绍了 Prometheus Operator 定义的几种自定义资源"
date: "2022-02-07"
menu: "main"
tags:
- "prometheus"
- "kubernetes"
categories:
- "technology"
---

## 设计

这篇文章介绍了 Prometheus Operator 的几种自定义资源 (CRD):

- Prometheus
- Alertmanager
- ThanosRuler
- ServiceMonitor
- PodMonitor
- Probe
- PrometheusRule
- AlertmanagerConfig

## Prometheus

它定义了在 Kubernetes 集群中安装 Prometheus 的方式. 它提供了一些配置项, 比如副本数、持久卷还有接收告警的 Alertmanagers.

对于每一个 Prometheus 资源, Operator 会在同一 namespace 中部署一个经过正确配置的 StatefulSet. 其配置文件 `secret/prometheus-name` 会被挂载到它的 pod 上.

它明确了 Prometheus 实例选择 ServiceMonitors 所用的标签. 任何时候, 一旦 ServiceMonitors 和 Prometheus 资源有更改, Operator 会根据他们生成重新配置文件, 然后更新到上述 Secret 配置文件中.

如果没有提供选择 ServiceMonitors 所用标签, 用户也可以自己管理 Secret, 同时该 Secret 也享受 Operator 配置安装 Prometheus 的能力.

## Alertmanager

该资源定义了在 Kubernetes 中配置安装 Alertmanager 的方式. 它提供了配置副本数和持久卷的选项.

对于每一个 Alertmanager 资源, Operator 会在同一个 namespace 中启动一个正确配置的 StatefulSet. Alertmanager pods 会挂载一个 `secret/alertmanager-name`, 它包含一个键为 `alertmanager.yaml` 的配置文件.

当配置了两个以上的副本, Operator 会启动 Alertmanager 高可用模式.

## ThanosRuler

该资源定义了 Kubernetes 中如何配置安装 Thanos Ruler. 借助 Thanos Ruler, 记录和告警规则可以被多个 Prometheus 实例处理.

一个 ThanosRuler 实例要求至少一个 queryEndpoint, 它指向 Thanos 查询器 或者 Prometheus 实例的位置. queryEndpoints 会配置 Thanos 运行时的 `--query` 参数. 了解 [Thanos](https://github.com/thanos-io/thanos).

## ServiceMonitor

该资源允许以声明方式定义应如何监视一组动态 Service, 使用标签来选择哪些 Service 需要监控. 这是一种约定, 来定义服务如何公开指标, 按照该约定 Prometheus 会自动发现新服务, 而不需要重新配置.

对于 Prometheus 监控的任何程序在 Kubernetes 中都要求有对应的 Endpoints 对象. Endpoints 对象本质上是 IP 地址列表. 通常，Endpoints 对象由 Service 对象填充。 Service 对象通过标签选择器发现 Pod 并将其添加到 Endpoints 对象中。

一个 Service 可能会暴露一个或者多个端口, 通常，它会生成一个 endpoints 列表, 指向一个具体的 Pod, 这和 kubernetes 中的 Endpoints 一样. ServiceMonitor 会自动发现 `Endpoints` 对象并配置 Prometheus 来监控这些 Pods.

`ServiceMonitorSpec` 中的 `endpoints` 部分, 是用来配置 Endpoints 被刮取指标数据时所需要的端口和参数. 对于一些高级用例, 要监控的后端 Pods 端口, 可能不是 service 中定义的. 因此在 endpoints 中定义 `endpoint` 时, 它必填.

> 注意: endpoints (小写) 是 ServiceMonitor 中的一个字段, 但 Endpoints (大写) 是 Kubernetes 中的对象.

ServiceMonitors 和发现的 targets 一样都可以来自任何 namespace. 这对于跨 namespace 的监控是很重要的, 比如 meta-monitoring (元监控). 使用 `PrometheusSpec/ServiceMonitorNamespaceSelector` 字段, 可以限制选择 ServiceMonitors 的命名空间. 使用 `ServiceMonitorSpec/namespaceSelector` 字段, 可以限制发现 Endpoints 对象的命名空间. 要发现所有 namespaces 的 targets 需要把 namespaceSelector 设置为空:

```yaml
spec:
  namespaceSelector:
    any: true
```

## PodMonitor

该资源定义了需要监控的一组动态的 pods. 它使用配置的标签选择要被监控的 pod. 它定义了一种指标暴露的方式, 使用这种方式, 新产生的 pod 就可以被自动发现, 不需要重新配置 prometheus.

Pod 是一个或多个容器的集合, 它可以在一个或多个端口上暴露 Prometheus 指标. `PodMonitor` 对象可以用来发现这些 pods, 并且为了监控他们, 会生成相应的 Prometheus 配置. `PodMonitorSpec/PodMetricsEndpoints` 定义了获取指标所用的端口和参数.

`PodMonitors` 和发现的 targets 可以来自任何 namespaces. 这对于跨 namespace 的监控是很重要的, 比如 meta-monitoring (元监控). 使用 `PodMonitorSpec/namespaceSelector`, 可以限制发现 Pods 对象的命名空间. 要发现所有 namespaces 的 targets 需要把 namespaceSelector 设置为空:

```yaml
spec:
  namespaceSelector:
    any: true
```

## Probe

该资源允许以声明的方式定义如何监控的 ingresses 和静态 targets. 除了 target, 它还要求一个探针, 这个探针是一个监控服务, 能监控 target 而且还能给 Prometheus 提供指标. 例如, 可以通过 [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter/) 来实现.

## PrometheusRule

该资源以声明方式定义一个或多个 Prometheus 实例使用的所需的 Prometheus 规则. 告警和记录规则可以以 YAML 的形式被保存和应用, 而且是热加载的, 不需要重启.

## AlertmanagerConfig

该资源定义了 Alertmanager 配置文件的子配置, 可以配置告警的接受者以及抑制策略, 它是 namespace 级别的. [这里](https://github.com/prometheus-operator/prometheus-operator/blob/main/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagerconfigs.yaml)是个例子.

> 注意: 该资源目前还不是 `stable`