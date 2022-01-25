---
weight: 1
title: "配置"
---

# 配置

该项目的配置修改都是在配置文件中完成的，配置文件在启动应用的时候通过`--config=`标签进行配置。比如：`oauth2 --config=/etc/oauth2/app.yaml`

配置文件介绍如下：

```yaml
# session 相关配置
session:
  name: session_id
  secret_key: "kkoiybh1ah6rbh0"
  # 过期时间
  # 单位秒
  # 默认20分钟
  max_age: 1200

# 数据库相关配置
# 默认是 default 连接
# 这里可以添加多个连接支持
db:
  default:
    type: mysql
    host: string
    port: 3306
    user: 123
    password: abc
    dbname: oauth2

# 可选
# redis 相关配置
# 可以提供:
# - 统一回话存储
# - oauth2 client 存储
redis:
  default:
    addr: 127.0.0.1:6379
    password: 
    db: 0

# oauth2 相关配置
oauth2:
  # access_token 过期时间
  # 单位小时
  # 默认2小时
  access_token_exp: 2
  # 签名 jwt access_token 时所用 key
  jwt_signed_key: "k2bjI75JJHolp0i"
  
  # oauth2 客户端配置
  # 数组类型
  # 可配置多客户端
  client:

      # 客户端id 必须全局唯一
    - id: test_client_1
      # 客户端 secret
      secret: test_secret_1
      # 应用名 在页面上必要时进行显示
      name: 测试应用1
      # 客户端 domain
      # !!注意 http/https 不要写错!!
      domain: http://localhost:9093
      # 权限范围
      # 数组类型
      # 可以配置多个权限 
      # 颁发的 access_token 中会包含该值 资源方可以对该值进行验证
      scope:
          # 权限范围 id 唯一
        - id: all
          # 权限范围名称
          # 会在页面（登录页面）进行展示
          title: "用户账号、手机、权限、角色等信息"

```