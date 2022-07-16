---
title: "Kubernetes 服务器配置和规划建设要求"
description: ""
summary: ""
date: "2022-07-16"
menu: "main"
draft: false
tags:
- kubernetes
categories:
- "technology"
---

新建集群的第一步就是要规划服务器、网络、操作系统等等, 下面就结合我平时的工作经验总结下相关的要求, 内容根据日常工作持续补充完善:

## 服务器配置

kubernetes 集群分为控制节点和数据节点, 它们对于配置的要求有所不同:

### 控制面

|节点规模|Master规格|
|-|-|
|1~5个节点|4核 8Gi（不建议2核 4Gi）|
|6~20个节点|	4核 16Gi|
|21~100个节点|8核 32Gi|
|100~200个节点|16核 64Gi|

系统盘40+Gi，用于储存 etcd 信息及相关配置文件等

### 数据面

- 规格：CPU >= 4核, 内存 >= 8Gi
- 确定整个集群的日常使用的**总核数**以及**可用度的容忍度**
  - 例如：集群总的核数有160核, 可以容忍10%的错误. 那么最小选择10台16核VM, 并且高峰运行的负荷不要超过 `160*90%=144核`. 如果容忍度是20%, 那么最小选择5台32核VM, 并且高峰运行的负荷不要超过`160*80%=128核`. 这样就算有一台VM出现故障, 剩余VM仍可以支持现有业务正常运行.
- 确定 `CPU:Memory` 比例. 对于使用内存比较多的应用, 例如Java类应用, 建议考虑使用1:8的机型
- 比如: `virtual machine 32C 64G 200G系统盘 数据盘可选`

### 什么情况下使用裸金属服务器?

- 集群日常规模能够达到1000核。一台服务器至少96核，这样可以通过10台或11台服务器即可构建一个集群。
- 快速扩大较多容器。例如：电商类大促，为应对流量尖峰，可以考虑使用裸金属来作为新增节点，这样增加一台裸金属服务器就可以支持很多个容器运行。

## 操作系统

- 建议安装 ubuntu 18.04/debian buster/ubuntu 20.04/centos 7.9 优先级从高到低
- linux kenerl 4.17+
- 安装 ansible v2.9 & python-netaddr 以运行 ansible 命令
- 安装 jinja 2.11+ 以运行 ansible playbooks
- 允许 IPv4 forwarding
- 部署节点的 ssh key 拷贝到所有节点
- 禁用防火墙

## 网络

- 单一集群采用统一的网卡命名比如:`eth0`等, 保证名称唯一
- 没有特殊要求, 服务器要求可访问外网

## 集群规模限制

- 每节点不超过 110 pods
- 不超过 5k nodes
- 总计不超过 15w pods
- 总计不超过 30w containers

## 参考

- [优刻得容器云UK8S集群节点配置推荐](https://docs.ucloud.cn/uk8s/introduction/node_requirements)
- [阿里云ACK容器服务之ECS选型](https://help.aliyun.com/document_detail/98886.html)