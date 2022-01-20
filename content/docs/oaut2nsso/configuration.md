---
weight: 1
title: "配置"
---

1. implicit 和 client credentials 模式是不会生成refresh token的, 刷新token时会删除原有的token重新发布新的token.
2. 每一种模式的配置详情如下:

```
var (
  DefaultCodeExp = time.Minute * 10
  DefaultAuthorizeCodeTokenCfg = &Config{AccessTokenExp: time.Hour * 2, RefreshTokenExp: time.Hour * 24 * 3, IsGenerateRefresh: true}
  DefaultImplicitTokenCfg = &Config{AccessTokenExp: time.Hour * 1}
  DefaultPasswordTokenCfg = &Config{AccessTokenExp: time.Hour * 2, RefreshTokenExp: time.Hour * 24 * 7, IsGenerateRefresh: true}
  DefaultClientTokenCfg = &Config{AccessTokenExp: time.Hour * 2}
  DefaultRefreshTokenCfg = &RefreshingConfig{IsGenerateRefresh: true, IsRemoveAccess: true, IsRemoveRefreshing: true}
)
```