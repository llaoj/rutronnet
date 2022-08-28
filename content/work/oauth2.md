---
title: 'Fall Plaza'
date: 2018-11-18T12:33:46+10:00
draft: false
weight: 1
heroHeading: 'Fall Plaza'
heroSubHeading: 'Revitalising a public space in Spain.'
heroBackground: 'work/work1.jpg'
thumbnail: 'work/work1-thumbnail.jpg'
images: [
	'https://source.unsplash.com/random/400x600/?nature', 
	'https://source.unsplash.com/random/400x300/?travel',
	'https://source.unsplash.com/random/400x300/?architecture',
	'https://source.unsplash.com/random/400x600/?buildings',
	'https://source.unsplash.com/random/400x300/?city',
	'https://source.unsplash.com/random/400x600/?business'
]
---

[llaoj/oauth2nsso](https://github.com/llaoj/oauth2nsso) 项目是基于 go-oauth2 打造的**独立**的 OAuth2.0 和 SSO 服务，提供了开箱即用的 OAuth2.0服务和单点登录SSO服务。开源一年多，获得了社区很多用户的关注，该项目多公司线上在用，其中包含上市公司。轻又好用，很稳。

感谢:
![sponsors](https://raw.githubusercontent.com/llaoj/oauth2nsso/master/docs/sponsors.png)

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
