---
title: "在istio service mesh中使用nginx反向代理"
description: ""
summary: ""
date: "2022-08-08"
menu: "main"
bookToC: true
draft: false
tags:
- istio
categories:
- "technology"
---

nginx反向代理的请求, 和我们直接请求有一定的区别, 比如:

## http version

nginx proxy 发出的反向代理请求的http version默认是: 1.0, 但是istio支持1.1 & 2.0, 所以如果不增加http版本限制的话istio就无法进行报文解析, 也就无法应用istio-proxy(sidecar)L7层代理策略, 我们知道istio流量治理是基于L7层的.

## http header: Host

有时候nginx发出的代理请求的http header中host的值, 不能保证是上游服务的host name. 在这种情况下, 是没办法匹配上游服务在istio-proxy中的L7流量治理的配置.

## 怎么解决?

所以, 需要在nginx代理配置处增加两项配置:

```nginx
...
    location / {
       proxy_http_version 1.1;                 <-
       proxy_set_header Host <upstream-host>;  <-
       proxy_pass http://<upstream-host>:8080;
    }
...
```

即可.

## 参考

- [nginx官方文档proxy_http_version介绍](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version)
- [nginx官方文档proxy_set_header介绍](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header)