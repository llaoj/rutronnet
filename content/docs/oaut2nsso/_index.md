---
weight: 3
bookCollapseSection: true
title: "OAuth2和SSO"
bookFlatSection: true
---

## llaoj/oauth2
1. OAuth2.0 Server: based on go-oauth2
2. SSO Server: based on the OAuth2 service
3. 多公司线上在用，其中包含上市公司。轻又好用。稳的一P。

## B站视频讲解

 [教你构建OAuth2.0和SSO单点登录服务(基于go-oauth2)](https://www.bilibili.com/video/BV1UA411v73P)

## 演示授权码(authorization_code)流程 & 单点登录(SSO)

![authorization_code_n_sso](./docs/demo-pic/authorization_code_n_sso.gif "authorization_code_n_sso")

## 目录

**实现了oauth2的四种工作流程**

1. authorization_code
2. implicit
3. password
4. client credentials

**其他**

5. 验证access_token (资源端)
6. 刷新token
7. 专门为SSO开发的logout
8. 配置说明

---


