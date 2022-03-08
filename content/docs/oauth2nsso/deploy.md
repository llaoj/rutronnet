---
weight: 3
title: "部署"
---

# 部署

## 修改配置和完善代码

克隆到代码之后，首先需要进行配置文件的修改和部分代码逻辑的编写：

```sh
# 克隆源码
git clone git@github.com:llaoj/oauth2nsso.git
cd oauth2nsso

# 根据实际情况修改配置
cp config.example.yaml /etc/oauth2nsso/config.yaml
vi /etc/oauth2nsso/config.yaml
...

# 如果使用 LDAP方式 验证用户, 直接修改配置文件即可
# OR
# 如果使用 数据库方式 验证用户, 需要修改源码
# 主要修改登录部分逻辑:
# 文件: model/user.go:21
# 方法: Authentication()
...
```

## 使用docker部署

**[推荐]** 容器化部署比较方便进行大规模部署，是当下的趋势。需要本地有 docker 环境。

```sh
# 构建镜像
docker build -t <image:tag> .

# 运行
docker run --rm --name=oauth2nsso --restart=always -d \
-p 9096:9096 \
-v <path to config.yaml>:/etc/oauth2nsso/config.yaml \
<image:tag>
```

## 基于源码部署

```sh
# 在仓库根目录
# 编译
go build -mod=vendor

# 运行
./oauth2nsso -config=/etc/oauth2nsso/config.yaml
```
