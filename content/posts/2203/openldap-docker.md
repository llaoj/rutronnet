---
title: "容器化部署 openldap"
description: "openldap 容器化安装部署"
date: "2022-03-03"
menu: "main"
tags:
- "ldap"
- "golang"
categories:
- "technology"
---

使用容器化安装非常便捷, 参考[osixia/openldap](https://github.com/osixia/docker-openldap)仓库使用说明安装即可, 如下:

```sh
docker stop openldap && docker rm openldap && \
docker run --name openldap --detach \
    -p 389:389 \
    -p 636:636 \
    --env LDAP_ORGANISATION="Rutron Net" \
    --env LDAP_DOMAIN="rutron.net" \
    --env LDAP_ADMIN_PASSWORD="your-password" \
    --env LDAP_READONLY_USER=true \
    --env LDAP_TLS_VERIFY_CLIENT=try \
    --volume /data/openldap/data:/var/lib/ldap \
    --volume /data/openldap/slapd.d:/etc/ldap/slapd.d \
    --hostname ldap.rutron.net \
    osixia/openldap:1.5.0
```

好了, 现在该服务同时支持 ldap 和 ldaps 协议, 有一个初始化的账号 `readonly/readonly`, 可以使用了~