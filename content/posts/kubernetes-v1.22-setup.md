---
title: "使用 Kubespray 安装 kubernetes v1.22 的过程"
date: "2022-02-14"
menu: "main"
tags:
- "kubespray"
- "kubernetes"
categories:
- "技术"
---

本文使用 kubespray 容器部署 kubernetes v1.22.5, 提供了预先下载的离线软件包/容器镜像. 仅需要几步即可部署高可用集群. 所有离线文件都来自官方下载 kubespray 安装过程会进行软件包验证, 放心使用.

## 最终版本说明

|包|版本|说明|
|-|-|-|
|kubespray|quay.io/kubespray/kubespray:v2.18.0|容器化部署|
|kubernetes|v1.22.5||

## 使用 kubespray 要满足

- 禁用防火墙
- **重要:** 本文使用 kubespray 的容器环境部署, 为避免影响节点部署(特别是 Runtime 部署), 所以需要一台**独立于集群外的服务器**执行下面的命令, 这台服务器安装 docker 19.03+
- 目标服务器要允许 IPv4 转发, 如果要给 pods 和 services 用 IPv6, 目标服务器要允许 IPv6 转发.

注意: 下面配置是适合 kubespray 的配置, 实际配置取决于集群规模.

- Master
  - Memory: 1500 MB
- Node
  - Memory: 1024 MB

> 这里介绍了[大集群配置指南](https://kubernetes.io/docs/setup/best-practices/cluster-large/#size-of-master-and-master-components).

## 下载 kubespray:v2.18.0 镜像

因为国内网络限制, 将官方镜像 `quay.io/kubespray/kubespray:v2.18.0` 下载并传至阿里云 `registry.cn-beijing.aliyuncs.com/llaoj/kubespray:v2.18.0`. 该镜像已经安装配置好了依赖软件包以及 kubespray 项目源文件. 放心使用

```sh
# 使用 kubespray 提供的镜像
# 能让我们避免处理各依赖包的复杂的版本问题
docker run --rm -it \
  -v "${PWD}"/inventory/mycluster:/kubespray/inventory/mycluster \
  -v "${HOME}"/.ssh/id_rsa:/root/.ssh/id_rsa \
  registry.cn-beijing.aliyuncs.com/llaoj/kubespray:v2.18.0 bash

# 后面的命令均在容器内部执行
```

## 构建 inventory 文件

Ansible 可同时操作属于一个组的多台主机, **组和主机之间的关系**通过 inventory 文件配置. 默认的文件路径为 /etc/ansible/hosts, 除默认文件外, 还可以同时使用多个 inventory 文件, 也可以从动态源, 或云上拉取 inventory 配置信息.

```sh
# 先定义节点
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)

# 复制示例配置
cp -rfp inventory/sample/. inventory/mycluster

# 使用 inventory_builder 工具生成 ansible inventory 文件
CONFIG_FILE=inventory/mycluster/hosts.yaml \
  HOST_PREFIX=hde-ceno \
  python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

## 自定义部署

按照集群的规划, 按说明修改下面两个文件中的配置:

```sh
# 这个文件是可修改的配置
cat inventory/mycluster/group_vars/all/all.yml
# 这个文件是强制的/不可改的配置
cat inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

请认真阅读每一个配置, 其中有个关于 CIDR 的配置如下(非常重要), 贴出来供参考:

```yaml
# CNI 插件配置, 可选 cilium, calico, weave 或 flannel
kube_network_plugin: calico

# sevice CIDR 配置
# 不能与其他网络重叠
kube_service_addresses: 10.233.0.0/18

# pod CIDR 配置
# 不能与其他网络重叠
kube_pods_subnet: 10.233.64.0/18

# 配置内部网络节点大小
# 配置每个节点可分配的 ip 个数
# 注意: 每节点最大 pods 数也受 kubelet_max_pods 限制, 默认 110
# 例子1:
# 最高64个节点, 每节点最高 254 或 kubelet_max_pods(两个取最小的) 个 pods 
#  - kube_pods_subnet: 10.233.64.0/18
#  - kube_network_node_prefix: 24
#  - kubelet_max_pods: 110
# 例子2:
# 最高128个节点, 每节点最高 126 或 kubelet_max_pods(两个取最小的) 个 pods 
#  - kube_pods_subnet: 10.233.64.0/18
#  - kube_network_node_prefix: 25
#  - kubelet_max_pods: 110
kube_network_node_prefix: 24
```

> 所有配置文件基本不需要修改, 如非必须, 采用默认值即可. 但要了解每个配置项作用. 执行部署时, 通过使用 `-e` 来覆盖默认配置

## [可选] 离线文件

安装过程需要下载很多软件包/镜像, 由于国内网络限制, 这些内容我已经提前下载好了. 如果你网络也不好, 请先下载解压备用, 无需修改 kubespray 的代码, 简单快捷.

```sh
# 下载离线文件压缩包
wget https://rutron.oss-cn-beijing.aliyuncs.com/k8s-setup/kubernetes-v1.22.5-amd64-packages.tar.gz
tar -xvf kubernetes-v1.22.5-amd64-packages.tar.gz
````

## 执行部署

- [可选] 拷贝离线文件

由于这一步不依赖其他软件包, 在安装开始就执行. 将上面下载的离线文件拷贝到目标服务器`/tmp/releases/`文件夹中.

```sh
for ip in ${IPS[@]}; do scp -r kubernetes-v1.22.5-amd64-packages/* $ip:/tmp/releases/; done
```

- 执行部署

```sh
ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml
```

> 通过使用 `-e` 来覆盖默认配置


- [可选] 导入离线镜像

这一步依赖`ctr`工具, 所以部署过程中**下载镜像出错**时再操作, 这时候 `ctr` 工具已经安装了.

```sh
for ip in ${IPS[@]}; do ssh $ip "ls /tmp/releases/images | awk '{print \"ctr -n k8s.io i import\", \"/tmp/releases/images/\"\$1}' | sh -x"; done
```

至此, 完成~

## 离线文件列表

下面是所有的文件列表, 仅供参考:

```sh
# 文件
https://github.com/coreos/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
=> /tmp/releases/etcd-v3.5.0-linux-amd64.tar.gz
https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz
=> /tmp/releases/cni-plugins-linux-amd64-v1.0.1.tgz
https://storage.googleapis.com/kubernetes-release/release/v1.22.5/bin/linux/amd64/kubeadm
=> /tmp/releases/kubeadm-v1.22.5-amd64
https://storage.googleapis.com/kubernetes-release/release/v1.22.5/bin/linux/amd64/kubelet
=> /tmp/releases/kubelet-v1.22.5-amd64
https://storage.googleapis.com/kubernetes-release/release/v1.22.5/bin/linux/amd64/kubectl
=> /tmp/releases/kubectl-v1.22.5-amd64
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.22.0/crictl-v1.22.0-linux-amd64.tar.gz
=> /tmp/releases/crictl-v1.22.0-linux-amd64.tar.gz
https://github.com/opencontainers/runc/releases/download/v1.0.3/runc.amd64
=> /tmp/releases/runc
https://github.com/containerd/containerd/releases/download/v1.5.8/containerd-1.5.8-linux-amd64.tar.gz
=> /tmp/releases/containerd-1.5.8-linux-amd64.tar.gz
https://github.com/containerd/nerdctl/releases/download/v0.15.0/nerdctl-0.15.0-linux-amd64.tar.gz
=> /tmp/releases/nerdctl-0.15.0-linux-amd64.tar.gz
https://github.com/projectcalico/calicoctl/releases/download/v3.20.3/calicoctl-linux-amd64
=> /tmp/releases/calicoctl
https://github.com/projectcalico/calico/archive/v3.20.3.tar.gz
=> /tmp/releases/calico-v3.20.3-kdd-crds/v3.20.3.tar.gz

# 镜像
quay.io/calico/node:v3.20.3
quay.io/calico/cni:v3.20.3
quay.io/calico/pod2daemon-flexvol:v3.20.3
quay.io/calico/kube-controllers:v3.20.3
k8s.gcr.io/pause:3.3
k8s.gcr.io/coredns/coredns:v1.8.0
k8s.gcr.io/dns/k8s-dns-node-cache:1.21.1
k8s.gcr.io/cpa/cluster-proportional-autoscaler-amd64:1.8.5
k8s.gcr.io/kube-apiserver:v1.22.5
k8s.gcr.io/kube-controller-manager:v1.22.5
k8s.gcr.io/kube-scheduler:v1.22.5
k8s.gcr.io/kube-proxy:v1.22.5
docker.io/library/nginx:1.21.4
```