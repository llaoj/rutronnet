---
title: "My First Post"
date: 2022-01-05T09:12:51+08:00
draft: false
---

#### 问题描述

这是一段问题描述这是一段问题描述这是一段问题描述这是一段问题描述

#### 日志分析

```shell
2021-12-12 [error] 10.206.0.1 has error in log, line: 123
2021-12-12 [error] 10.206.0.1 has error in log, line: 123
2021-12-12 [error] 10.206.0.1 has error in log, line: 123
2021-12-12 [error] 10.206.0.1 has error in log, line: 123
```
以上就是测试的日志输出，说明了一种错误。

#### 代码分析

从代码 `examples/log.go:234` 分析得出，应该这么修改：

```golang
func BenchmarkOneRoute(B *testing.B) {
	router := New()
	router.GET("/ping", func(c *Context) {})
	runRequest(B, router, "GET", "/ping")
}
``` 

向请求中增加 header 参数

```yaml
nginx.ingress.kubernetes.io/configuration-snippet: |
    proxy_set_header "Authorization: Bearer $arg_token";
```

去掉 token 参数

```
rewrite ^(.*?)start=\d+\&?(.*)$ $1$2 last;
```

ingress-nginx-external-auth