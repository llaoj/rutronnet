---
weight: 2
title: "API列表"
---

# API列表

## 1 authorization_code

### 1-1 获取授权code

**请求方式**

`GET` `/authorize`

**参数说明**  

|参数|类型|说明|
|-|-|-|
|client_id|string|在oauth2 server注册的client_id,见配置文件[oauth2.client.id](http://rutron.net/docs/oauth2nsso/configuration/)|
|response_type|string|固定值:`code`|
|scope|string|权限范围,如:`str1,str2,str3`,str为配置文件中[oauth2.client.scope.id](http://rutron.net/docs/oauth2nsso/configuration/)的值 |
|state|string|表示客户端的当前状态,可以指定任意值,认证服务器会原封不动地返回这个值|
|redirect_uri|string|回调uri,会在后面添加query参数`?code=xxx&state=xxx`,发放的code就在其中|

**请求示例**

```sh
# 浏览器请求
http://localhost:9096/authorize?client_id=test_client_1&response_type=code&scope=all&state=xyz&redirect_uri=http://localhost:9093/cb

# 302跳转,返回code
http://localhost:9093/cb?code=XUNKO4OPPROWAPFKEWNZWA&state=xyz
```

### 1-2 使用`code`交换`token`

**请求方式**

`POST` `/token`

**请求头 Authorization**

- basic auth
- username: `client_id`
- password: `client_secret`

**Header**  
`Content-Type: application/x-www-form-urlencoded`

**Body参数说明**  

|参数|类型|说明|
|-|-|-|
|grant_type|string|固定值`authorization_code`|
|code|string| 1-1 发放的code|
|redirect_uri|string| 1-1 填写的redirect_uri|

**Response返回示例**  

```json
{
    "access_token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiIyMjIyMjIiLCJleHAiOjE1ODU3MTU1NTksInN1YiI6InRlc3QifQ.ZMgIDQMW7FGxbF1V8zWOmEkmB7aLH1suGYjhDdrT7aCYMEudWUoiCkWHSvBmJahGm0RDXa3IyDoGFxeMfzlDNQ",
    "expires_in": 7200,
    "refresh_token": "JG7_WGLWXUOW2KV2VLJKSG",
    "scope": "all",
    "token_type": "Bearer"
}
```

## 2 implicit

资源请求方(client方)使用, 
多用于没有后端的应用, 
用户授权登录之后, 会直接向前端发送令牌(`access_token`)

**请求方式**

`GET` `/authorize`

**参数说明**  

|参数|类型|说明|
|-|-|-|
|client_id|string|在 oauth2 server 注册的client_id|
|response_type|string|固定值`token`|
|scope|string|权限范围,同1-1中说明|
|state|string|验证请求的标志字段|
|redirect_uri|string|发放`code`用的回调uri,回调时会在uri后面跟上`?code=**&state=###`|

**请求示例**

```sh
http://localhost:9096/authorize?client_id=test_client_1&response_type=token&scope=all&state=xyz&redirect_uri=http://localhost:9093/cb
```

**返回示例**

```sh
# 302 跳转,返回 access_token
http://localhost:9093/cb#access_token=eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ0ZXN0X2NsaWVudF8xIiwiZXhwIjoxNTkxNDI3OTMwLCJzdWIiOiJhZG1pbiJ9.RBYns9UnNYDHINSBzvHWHRzuKCpzKmsxUnKt30lntmGvXmVDoByZtlB0RHAVB59PHBlJNO_YUBZzC2odwCa8Tg
&expires_in=3600&scope=all&state=xyz&token_type=Bearer
```

**注意**

1. 这里会返回请求时设置的`state`, 请在进行下一步之前验证它, 防止请求被劫持或者篡改
2. 这种方式把令牌直接传给前端，是很不安全的。因此，只能用于一些安全要求不高的场景，并且令牌的有效期必须非常短，通常就是会话期间（session）有效，浏览器关掉，令牌就失效了


## 3 password

资源请求方(client方)使用
如果充分信任接入应用(client), 用户就可以直接把用户名密码给接入应用.
接入应用使用用户账号密码申请令牌.

**请求方式**

`POST` `/token`

**请求头 Authorization**

- basic auth
- username: `client_id`
- password: `client_secret`

**Header**  
`Content-Type: application/x-www-form-urlencoded`

**Body参数说明**  

|参数|类型|说明|
|-|-|-|
|grant_type|string|固定值`password`|
|username|string|用户名|
|password|string|用户密码|
|scope|string|权限范围,同1-1中说明|

**返回示例**  

```json
{
    "access_token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ0ZXN0X2NsaWVudF8xIiwiZXhwIjoxNTkxNDMyNzA3LCJzdWIiOiJhZG1pbiJ9.ECfUkCMUZE8I6GH3XTDcJnQgDryiRyyBhEHBW-dCxzFWaR-mvU5dsx3XV2bx-LWZzPJTBAQ3rB5QOb4BHjnBXw",
    "expires_in": 7200,
    "refresh_token": "AH-B00RKXTME9WXDPSBSTG",
    "scope": "all",
    "token_type": "Bearer"
}
```

## 4 client_credentials

资源请求方(client方)使用
使用在oauth2服务器注册的client_id 和 client_secret 获取 access_token,
发出 API 请求时，它应将access_token作为 Bearer 令牌传递到 Authorization 请求头中。

**请求方式**

`POST` `/token`

**请求头 Authorization**

- basic auth
- username: `client_id`
- password: `client_secret`

**Header**  

`Content-Type: application/x-www-form-urlencoded`

**Body参数说明**  

|参数|类型|说明|
|-|-|-|
|grant_type|string|固定值`client_credentials`|
|scope|string|权限范围,同1-1中说明|

**返回示例**  

```json
{
    "access_token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJlbWJlZGVkLWg1LWFwaSIsImV4cCI6MTU4OTk3NzQyNn0.Pu93fy0-gyiFqExBkCFAKTVJ1on_RpOSexzkHqczA6n6kB2_mOHbTMOyGK_Di7bHxZ3JqpZeyDoKQBtUe_T7jw",
    "expires_in": 7200,
    "token_type": "Bearer"
}
```

## 5 验证token

**接口说明**

这个接口是资源端使用的, 
用来验证 `access_token` `scope` 和 `domain` .

**请求方式**

`GET`  `/verify`

**请求头 Authorization**

- Bearer Token
- Token: `access_token`

**返回示例**  

正确 Status Code: 200

Response Body:

```json
{
  "client_id": "test_client_1",
  "domain": "http://127.0.0.1:9093",
  "expires_in": 7188,
  "scope": "all",
  "user_id": ""
}
```
> **注意:** 接口还会一起返回权限范围`scope` 和 client 的注册 domain, 这里推荐验证下, 请求方的身份和权限

错误 Status Code: 400

Response Body: `invalid access token`

## 6 刷新token

刷新access_token, 使用refresh_token换取access_token

**请求方式**

`POST` `/token`

**请求头 Authorization**

- basic auth
- username: `client_id`
- password: `client_secret`

**Header**

`Content-Type: application/x-www-form-urlencoded`

**Body参数说明**


|参数|类型|说明|
|-|-|-|
|grant_type|string|固定值`refresh_token`|
|refresh_token|string|之前获取的refresh_token|

**返回示例**

```json
{
    "access_token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiIyMjIyMjIiLCJleHAiOjE1ODU4MTc2MTMsInN1YiI6IjEifQ.yNpQIbklhtsSr5KEkJMAR4I30c85OEriYwAOpL_ukRBJ1qsSziT05HFN-kxVN1-qM18TzVEf8beCvugyhpgpsg",
    "expires_in": 7200,
    "refresh_token": "2AH_LQHPUYK8XML4LKMQKG",
    "scope": "all",
    "token_type": "Bearer"
}
```

## 7 logout

专门为SSO开发, 
主要是销毁浏览器的会话, 退出登录状态, 跳转到指定链接(redirect_uri)

**请求方式**

`GET` `/logout?redirect_uri=xxx`

**参数说明**  

|参数|类型|说明|
|-|-|-|
|redirect_uri|string|退出登录后跳转到的地址,建议使用1-1所生成的地址, 需要urlencode|

**请求示例**  

```sh
http://localhost:9096/logout?redirect_uri=http%3a%2f%2flocalhost%3a9096%2fauthorize%3fclient_id%3dtest_client_1%26response_type%3dcode%26scope%3dall%26state%3dxyz%26redirect_uri%3dhttp%3a%2f%2flocalhost%3a9093%2fcb
```