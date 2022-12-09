---
title: 'OAuth2&SSO'
date: "2020-10-01 12:00"
draft: false
weight: 1
heroHeading: 'OAuth2&SSO'
heroSubHeading: '基于Golang打造的独立的OAuth2.0和SSO服务'
heroBackground: 'work/sso-authentication-bg.png'
thumbnail: 'work/sso-authentication.svg'
images: []
---

项目介绍

[llaoj/oauth2nsso](https://github.com/llaoj/oauth2nsso) 项目是基于 go-oauth2 打造的**独立**的 OAuth2.0 和 SSO 服务，提供了开箱即用的 OAuth2.0服务和单点登录SSO服务。开源一年多，获得了社区很多用户的关注，该项目多公司线上在用，其中包含上市公司。轻又好用，很稳

实现了oauth2的四种工作流程, 他们分别是: 1. Authorization Code; 2. Implicit; 3. Password; 4. Client Credentials

同时提供了三个扩展功能, 分别是: 5. 资源端用的验证 access_token 接口 `/verify`; 6. 刷新 token 接口 `/refresh`; 7. 专门为 SSO 开发的客户端登出接口 `/logout`

感谢:
![sponsors](https://raw.githubusercontent.com/llaoj/oauth2nsso/master/docs/sponsors.png)