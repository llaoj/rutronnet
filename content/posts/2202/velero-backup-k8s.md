---
title: "使用 velero 备份 kubernetes 指引"
description: "velero 是一款优秀的 kubernetes 备份程序，它通过 apiserver api 进行资源的备份和恢复。"
date: "2022-02-02"
menu: "main"
tags:
- "backup"
- "kubernetes"
categories:
- "technology"
---

## 要求

- kubernetes 版本 1.7+，velero 的每个主版本对 kuberetes 的版本要求不同，详情请参考官方文档说明。[官方文档通道](https://velero.io/docs/v1.7/)
- velero 所在服务器有 kubectl 命令, 且能连上集群

**我们先从最简单的体验开始**

## 1. 安装 velero 客户端

下载二进制安装包, 点击 latest release, 下载 `velero-v1.7.0-linux-amd64.tag.gz` (以 release 页面为准), 解压

```sh
tar -xvf <RELEASE-TARBALL-NAME>.tar.gz
```

然后将二进制文件 velero 移动到 $PATH 中的一个目录, 如 `/usr/local/bin`

## 2. 创建 credentials

备份文件保存在对象存储中, 在当前目录下创建 credentials-velero 文件, 声明连接对象存储所用的账号密码

```sh
[default]
aws_access_key_id = <your key_id>
aws_secret_access_key = <your secret>
```

## 3. 安装 velero 服务端

通过 velero 客户端在 kubernetes 中安装 deployment/velero，velero 提供了很多 stroage provider, 能将备份文件存储到比如 aws, aliyun-oss 中, 他们大都是支持 s3 接口的. 下面这个例子使用 s3 接口兼容的对象存储:

```shell
BUCKET=<your bucket>
REGION=<your region>
S3URL=<your s3url>

velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.3.0 \
    --bucket $BUCKET \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=$REGION,s3ForcePathStyle="true",s3Url=$S3URL
```

## 4. 进行一次备份

```shell
velero backup create first-all-ns
```

查看所有的备份

```sh
velero backup get
```

## 5. 恢复指定的 namespace

```sh
velero restore create --from-backup <backup name> \
--include-namespaces <namespace name> \
--restore-volumes=false \
```
> 恢复时忽略存储卷，默认是true。根据自己实际情况配置。

## 6. 每日定时更新整个集群

```shell
velero schedule create all-daily --schedule="@daily"
```

至此体验结束, 下面是一些拓展内容

---

## 拓展内容

### 备份文件存在哪里?

- `BackupStorageLocations` : 用来存储 kubernetes 原数据, 包括各种资源的配置清单等。这个命令可以看到上面安装 velero 时自动创建的 BackupStorageLocations 资源

```shell
kubectl -n velero get BackupStorageLocations
```

- `VolumeSnapshotLocation` : 用来存储存储卷的数据。这个命令可以看到上面安装 velero 时自动创建的 VolumeSnapshotLocation 资源

```shell
kubectl -n velero get VolumeSnapshotLocation
```

### BackupStorageLocations

首先, 创建后端存储使用的密钥文件，在 velero namespace 下创建对接 `BackupStorageLocations` 使用的 secret

```sh
kubectl create secret generic -n velero credentials --from-file=bsl=</path/to/credentialsfile>
```

这里创建一个叫 credentials 的 secret, 键: `bsl`, 值: `</path/to/credentialsfile>`, 后面 velero 和 BackupStorageLocations 通讯时候就用这个 credentials，下面使用这个 secret 创建 BackupStorageLocations。

```sh
velero backup-location create <bsl-name> \
  --provider <provider> \
  --bucket <bucket> \
  --config region=<region> \
  --credential=<secret-name>=<key-within-secret>
```

下面这个命令可以查看新创建的 BackupStorageLocations 是否可以使用, 如果有 `Avaliable` 表示创建成功

```shell
velero backup-location get
```

当我们使用这个 BackupStorageLocations 进行备份的时候, 可以使用 `--storage-location` 标志, 如下

```shell
velero backup create --storage-location <bsl-name>
```

或者不使用 `--storage-location <bsl-name>` 标志, 直接将它设置为默认 BackupStorageLocations, 这样

```shell
velero backup-location set --default <bsl-name>
```

当然, 如果想更改 credetial, 可以重新创建一个 secret 然后使用下面命令更换 secret 即可,

```shell
velero backup-location set <bsl-name> \
  --credential=<secret-name>=<key-within-secret>
```

---

## 2021.12.22日补充

问题1: 我在使用 velero v1.1.0 备份一个经过二开的 kubernetes 集群， 发现每次执行 schedule 都会报错。

```sh
level=error msg="backup failed" controller=backup error="rpc error: code = Unknown desc = EOF,..."

logSource="pkg/controller/backup_controller.go:233"
```

google 发现有人遇到了这个问题，大概是内存不够导致通讯失败。 [参考 issue](https://github.com/vmware-tanzu/velero/issues/1986)，按照 @skriss 所说， 提高 deployment/velero limits.memory 问题就解决了， 我的配置是 `1024Mi`
