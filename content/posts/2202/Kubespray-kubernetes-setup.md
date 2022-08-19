---
title: "使用Kubespray安装kubernetes的教程"
date: "2022-02-14"
menu: "main"
tags:
- "kubespray"
- "kubernetes"
categories:
- "technology"
---

本文使用 kubespray 容器部署 kubernetes v1.22, 提供了从国外搬运的离线软件包/容器镜像. 仅需要几步即可部署高可用集群. 所有离线文件都来自官方下载 kubespray 安装过程会进行软件包验证, 放心使用.

## 前提

- 禁用防火墙
- **重要:** 本文使用 kubespray 的容器环境部署, 为避免影响节点部署(特别是 Runtime 部署), 所以需要一台**独立于集群外的服务器**执行下面的命令, 这台服务器安装 docker 19.03+ 并到所有节点SSH免密进入.
- 目标服务器要允许 IPv4 转发, 如果要给 pods 和 services 用 IPv6, 目标服务器要允许 IPv6 转发.

注意: 下面配置是适合 kubespray 的配置, 实际配置取决于集群规模.

- Master
  - Memory: 1500 MB
- Node
  - Memory: 1024 MB

## 文件和镜像搬运

由于国内网络的限制, 直接使用Kubespray是无法成功安装集群的. 所以我写了一个脚本, 将文件上传至阿里云OSS, 镜像上传至阿里云ACR. 下面是脚本的内容, 请在国外服务器上运行, 比如GitHub Actions、Google云控制台等.

```shell
#!/bin/bash

set -x
KUBSPRAY_VERSION=v2.18.1
OSS_ENDPOINT=<OSS ENDPOINT>
OSS_ACCESS_KEY_ID=<OSS ACCESS KEY ID>
OSS_ACCESS_KEY=<OSS ACCESS KEY>
OSS_CLOUD_URL=<Bucket和文件路径>
ACR_REPO=<ACR地址>
ACR_USERNAME=<ACR用户名>
ACR_PASSWORD=<ACR密码>
MY_IMAGE_REPO=${ACR_REPO}/<ACR命名空间>

wget -O kubespray-src.zip https://github.com/kubernetes-sigs/kubespray/archive/refs/tags/${KUBSPRAY_VERSION}.zip
unzip -q kubespray-src.zip
SRC_PATH=$(unzip -l kubespray-src.zip | sed -n '5p' | awk '{print $4}')

cd ${SRC_PATH}contrib/offline
./generate_list.sh -i inventory/sample/inventory.ini
cat temp/files.list
cat temp/images.list

echo "Download files and upload to OSS"
wget -qx -P temp/files -i temp/files.list
tree temp/
wget -q https://gosspublic.alicdn.com/ossutil/1.7.13/ossutil64 && chmod 755 ossutil64
./ossutil64 \
  -e $OSS_ENDPOINT \
  -i $OSS_ACCESS_KEY_ID \
  -k $OSS_ACCESS_KEY \
  cp temp/files/ oss://${OSS_CLOUD_URL} -ruf --acl=public-read

echo "Copy images to ACR"
cat >> temp/images.list <<EOF
quay.io/metallb/speaker:v0.10.3
quay.io/metallb/controller:v0.10.3
quay.io/kubespray/kubespray:v2.18.1
EOF
skopeo login -u $ACR_USERNAME -p $ACR_PASSWORD $ACR_REPO
for image in $(cat temp/images.list)
do 
  myimage=${image#*/}
  myimage=${MY_IMAGE_REPO}/${myimage/\//_}
  echo $myimage >> temp/myimages.list
  skopeo copy docker://${image} docker://${myimage}
done
cat temp/myimages.list
```

## 运行kubespray容器

直接使用 kubespray 提供的镜像能让我们避免处理各依赖包的复杂的版本问题

```sh
docker run --rm -it \
  -v ${PWD}/inventory/mycluster:/kubespray/inventory/mycluster \
  -v ${HOME}/.ssh/id_rsa:/root/.ssh/id_rsa \
  registry.cn-beijing.aliyuncs.com/llaoj/kubespray_kubespray:v2.18.1 bash

# 后面的命令均在容器内部执行
cp -r inventory/sample inventory/mycluster
```

## 编辑inventory文件

Ansible 可同时操作属于一个组的多台主机, **组和主机之间的关系**通过 inventory 文件配置. 修改`inventory/mycluster/inventory.ini`即可, 该文件已经配置好了供你修改的模版:

```
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
# node1 ansible_host=95.54.0.12  # ip=10.3.0.1 etcd_member_name=etcd1
# node2 ansible_host=95.54.0.13  # ip=10.3.0.2 etcd_member_name=etcd2
# node3 ansible_host=95.54.0.14  # ip=10.3.0.3 etcd_member_name=etcd3
# node4 ansible_host=95.54.0.15  # ip=10.3.0.4 etcd_member_name=etcd4
# node5 ansible_host=95.54.0.16  # ip=10.3.0.5 etcd_member_name=etcd5
# node6 ansible_host=95.54.0.17  # ip=10.3.0.6 etcd_member_name=etcd6

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
# node1
# node2
# node3

[etcd]
# node1
# node2
# node3

[kube_node]
# node2
# node3
# node4
# node5
# node6

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

## 自定义部署

按照集群的规划, 按说明修改下面两个文件中的配置:

```sh
vi inventory/mycluster/group_vars/all/*.yml
vi inventory/mycluster/group_vars/k8s_cluster/*.yml
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

> 所有配置文件基本不需要修改, 如非必须, 采用默认值即可. 但要了解每个配置项作用. 

## 更换文件/镜像地址

将我们上面搬运到国内的文件包/镜像的地址配置上, 并放在`inventory/mycluster/group_vars/all/`文件夹下:

```
vi inventory/mycluster/group_vars/all/files-images.yaml
```

将下面的内容是我搬运之后的地址, 你可以直接使用. 或者使用你的地址替换`oss_files_repo`和`acr_image_repo`两个地址

```yaml
# files
oss_files_repo: "https://rutron.oss-cn-beijing.aliyuncs.com/kubernetes"
kubelet_download_url: "{{ oss_files_repo }}/storage.googleapis.com/kubernetes-release/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubelet"
kubectl_download_url: "{{ oss_files_repo }}/storage.googleapis.com/kubernetes-release/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubectl"
kubeadm_download_url: "{{ oss_files_repo }}/storage.googleapis.com/kubernetes-release/release/{{ kubeadm_version }}/bin/linux/{{ image_arch }}/kubeadm"
etcd_download_url: "{{ oss_files_repo }}/github.com/coreos/etcd/releases/download/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
flannel_cni_download_url: "{{ oss_files_repo }}/github.com/flannel-io/cni-plugin/releases/download/{{ flannel_cni_version }}/flannel-{{ image_arch }}"
cni_download_url: "{{ oss_files_repo }}/github.com/containernetworking/plugins/releases/download/{{ cni_version }}/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"
calicoctl_download_url: "{{ oss_files_repo }}/github.com/projectcalico/calicoctl/releases/download/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
calico_crds_download_url: "{{ oss_files_repo }}/github.com/projectcalico/calico/archive/{{ calico_version }}.tar.gz"
crictl_download_url: "{{ oss_files_repo }}/github.com/kubernetes-sigs/cri-tools/releases/download/{{ crictl_version }}/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
helm_download_url: "{{ oss_files_repo }}/get.helm.sh/helm-{{ helm_version }}-linux-{{ image_arch }}.tar.gz"
runc_download_url: "{{ oss_files_repo }}/github.com/opencontainers/runc/releases/download/{{ runc_version }}/runc.{{ image_arch }}"
crun_download_url: "{{ oss_files_repo }}/github.com/containers/crun/releases/download/{{ crun_version }}/crun-{{ crun_version }}-linux-{{ image_arch }}"
kata_containers_download_url: "{{ oss_files_repo }}/github.com/kata-containers/kata-containers/releases/download/{{ kata_containers_version }}/kata-static-{{ kata_containers_version }}-{{ ansible_architecture }}.tar.xz"
gvisor_runsc_download_url: "{{ oss_files_repo }}/storage.googleapis.com/gvisor/releases/release/{{ gvisor_version }}/{{ ansible_architecture }}/runsc"
gvisor_containerd_shim_runsc_download_url: "{{ oss_files_repo }}/storage.googleapis.com/gvisor/releases/release/{{ gvisor_version }}/{{ ansible_architecture }}/containerd-shim-runsc-v1"
nerdctl_download_url: "{{ oss_files_repo }}/github.com/containerd/nerdctl/releases/download/v{{ nerdctl_version }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
krew_download_url: "{{ oss_files_repo }}/github.com/kubernetes-sigs/krew/releases/download/{{ krew_version }}/krew-{{ host_os }}_{{ image_arch }}.tar.gz"
containerd_download_url: "{{ oss_files_repo }}/github.com/containerd/containerd/releases/download/v{{ containerd_version }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"

# images
acr_image_repo: "registry.cn-beijing.aliyuncs.com/llaoj"
kube_image_repo: "{{ acr_image_repo }}"
netcheck_server_image_repo: "{{ acr_image_repo }}/mirantis_k8s-netchecker-server"
netcheck_agent_image_repo: "{{ acr_image_repo }}/mirantis_k8s-netchecker-agent"
etcd_image_repo: "{{ acr_image_repo }}/coreos_etcd"
cilium_image_repo: "{{ acr_image_repo }}/cilium_cilium"
cilium_init_image_repo: "{{ acr_image_repo }}/cilium_cilium-init"
cilium_operator_image_repo: "{{ acr_image_repo }}/cilium_operator"
multus_image_repo: "{{ acr_image_repo }}/k8snetworkplumbingwg_multus-cni"
flannel_image_repo: "{{ acr_image_repo }}/coreos_flannel"
calico_node_image_repo: "{{ acr_image_repo }}/calico_node"
calico_cni_image_repo: "{{ acr_image_repo }}/calico_cni"
calico_flexvol_image_repo: "{{ acr_image_repo }}/calico_pod2daemon-flexvol"
calico_policy_image_repo: "{{ acr_image_repo }}/calico_kube-controllers"
calico_typha_image_repo: "{{ acr_image_repo }}/calico_typha"
weave_kube_image_repo: "{{ acr_image_repo }}/weaveworks_weave-kube"
weave_npc_image_repo: "{{ acr_image_repo }}/weaveworks_weave-npc"
kube_ovn_container_image_repo: "{{ acr_image_repo }}/kubeovn_kube-ovn"
kube_router_image_repo: "{{ acr_image_repo }}/cloudnativelabs_kube-router"
pod_infra_image_repo: "{{ acr_image_repo }}/pause"
install_socat_image_repo: "{{ acr_image_repo }}/xueshanf_install-socat"
nginx_image_repo: "{{ acr_image_repo }}/library_nginx"
haproxy_image_repo: "{{ acr_image_repo }}/library_haproxy"
coredns_image_repo: "{{ acr_image_repo }}/coredns_coredns"
nodelocaldns_image_repo: "{{ acr_image_repo }}/dns_k8s-dns-node-cache"
dnsautoscaler_image_repo: "{{ acr_image_repo }}/cpa_cluster-proportional-autoscaler-amd64"
registry_image_repo: "{{ acr_image_repo }}/library_registry"
metrics_server_image_repo: "{{ acr_image_repo }}/metrics-server_metrics-server"
addon_resizer_image_repo: "{{ acr_image_repo }}/addon-resizer"
local_volume_provisioner_image_repo: "{{ acr_image_repo }}/sig-storage_local-volume-provisioner"
cephfs_provisioner_image_repo: "{{ acr_image_repo }}/external_storage_cephfs-provisioner"
rbd_provisioner_image_repo: "{{ acr_image_repo }}/external_storage_rbd-provisioner"
local_path_provisioner_image_repo: "{{ acr_image_repo }}/rancher_local-path-provisioner"
ingress_nginx_controller_image_repo: "{{ acr_image_repo }}/ingress-nginx_controller"
alb_ingress_image_repo: "{{ acr_image_repo }}/amazon_aws-alb-ingress-controller"
cert_manager_controller_image_repo: "{{ acr_image_repo }}/jetstack_cert-manager-controller"
cert_manager_cainjector_image_repo: "{{ acr_image_repo }}/jetstack_cert-manager-cainjector"
cert_manager_webhook_image_repo: "{{ acr_image_repo }}/jetstack_cert-manager-webhook"
csi_attacher_image_repo: "{{ acr_image_repo }}/sig-storage_csi-attacher"
csi_provisioner_image_repo: "{{ acr_image_repo }}/sig-storage_csi-provisioner"
csi_snapshotter_image_repo: "{{ acr_image_repo }}/sig-storage_csi-snapshotter"
snapshot_controller_image_repo: "{{ acr_image_repo }}/sig-storage_snapshot-controller"
csi_resizer_image_repo: "{{ acr_image_repo }}/sig-storage_csi-resizer"
csi_node_driver_registrar_image_repo: "{{ acr_image_repo }}/sig-storage_csi-node-driver-registrar"
cinder_csi_plugin_image_repo: "{{ acr_image_repo }}/k8scloudprovider_cinder-csi-plugin"
aws_ebs_csi_plugin_image_repo: "{{ acr_image_repo }}/amazon_aws-ebs-csi-driver"
dashboard_image_repo: "{{ acr_image_repo }}/kubernetesui_dashboard-amd64"
dashboard_metrics_scraper_repo: "{{ acr_image_repo }}/kubernetesui_metrics-scraper"
metallb_speaker_image_repo: "{{ acr_image_repo }}/metallb_speaker"
metallb_controller_image_repo: "{{ acr_image_repo }}/metallb_controller"
```

## 执行部署

在容器内执行命令:

### 安装集群

```sh
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml
```

### 扩充节点

修改`inventory/mycluster/inventory.ini`文件, 添加好节点并执行:

```shell
ansible-playbook -i inventory/mycluster/inventory.ini scale.yml
```

至此, 完成~

## 问题列表

1. 报错: No package matching 'container-selinux' found available, installed or updated

```
fatal: [hde-ceno1]: FAILED! => {"attempts": 4, "changed": false, "msg": "No package matching 'container-selinux' found available, installed or updated", "rc": 126, "results": ["libselinux-python-2.5-15.el7.x86_64 providing libselinux-python is already installed", "7:device-mapper-libs-1.02.170-6.el7.x86_64 providing device-mapper-libs is already installed", "nss-3.44.0-7.el7_7.x86_64 providing nss is already installed", "No package matching 'container-selinux' found available, installed or updated"]}
```

解决办法: [打开链接](http://mirror.centos.org/centos/7/extras/x86_64/Packages/)找到服务器所需要的 container-selinux 包, 选择最新的包复制链接地址, 在问题机器上执行:

```sh
yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.119.2-1.911c772.el7_8.noarch.rpm
```

> 替换命令中的 centos 包地址即可, 执行完毕之后再次执行部署.