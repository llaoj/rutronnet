---
weight: 1
bookCollapseSection: false
title: "OAuth2&SSO"
---

# OAuth2&SSO

## 项目介绍

[llaoj/oauth2](https://github.com/llaoj/oauth2) 项目是基于 go-oauth2 打造的**独立**的 OAuth2.0 和 SSO 服务，提供了开箱即用的 OAuth2.0服务和单点登录SSO服务。开源一年多，获得了社区很多用户的关注，该项目多公司线上在用，其中包含上市公司。轻又好用，稳的一P。

## B站视频讲解

 [教你构建OAuth2.0和SSO单点登录服务(基于go-oauth2)](https://www.bilibili.com/video/BV1UA411v73P)

## 动图演示

授权码(authorization_code)流程 & 单点登录(SSO)

![authorization_code_n_sso](https://raw.githubusercontent.com/llaoj/oauth2/master/docs/demo-pic/authorization_code_n_sso.gif)

## 主要功能

**实现了oauth2的四种工作流程**

1. authorization_code
2. implicit
3. password
4. client credentials

**扩展功能**

5. 资源端用的验证 access_token 接口 `/verify`
6. 刷新 token 接口 `/refresh`
7. 专门为 SSO 开发的客户端登出接口 `/logout`

详情见[API说明](/docs/oaut2nsso/apis/)