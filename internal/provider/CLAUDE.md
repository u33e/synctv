# [CLAUDE.md](../../CLAUDE.md) > internal/provider 模块

> [!NOTE]
> 面包屑: [SyncTV 项目根](../../CLAUDE.md) / [internal](../) / provider (OAuth2 提供商)

## 模块概述

`internal/provider` 是 SyncTV 的 OAuth2 认证系统，支持多种第三方登录方式，包括官方提供商、聚合提供商和插件系统。

## 目录结构

```
internal/provider/
├── provider.go       # Provider 接口定义
├── aggregation.go   # 聚合 Provider 定义
├── providers/       # 官方提供商实现
├── aggregations/    # 聚合提供商 (多账号登录)
└── plugins/         # 插件系统
```

## 核心接口

### Provider 接口
**文件**: [provider.go](provider.go)

```go
type Provider interface {
    // 获取提供商类型
    Type() string
    
    // 获取 OAuth2 配置
    Config() *oauth2.Config
    
    // 获取用户信息
    GetUser(ctx context.Context, token *oauth2.Token) (any, error)
}
```

### Aggregation 接口
**文件**: [aggregation.go](aggregation.go)

```go
type Aggregation interface {
    // 获取聚合类型
    Type() string
    
    // 列出所有关联账号
    List() ([]*AggregationUser, error)
    
    // 添加账号
    Add(user *AggregationUser) error
    
    // 删除账号
    Delete(id string) error
}
```

## 支持的提供商

### 官方 Providers
位置: [providers/](providers/)

支持的提供商类型:
- GitHub
- Google
- Microsoft
- QQ
- 微信
- 抖音
- Gitee
- GitLab

### 聚合 Aggregations
位置: [aggregations/](aggregations/)

允许用户将多个第三方账号关联到同一个 SyncTV 账号。

### 插件 Plugins
位置: [plugins/](plugins/)

支持通过插件方式扩展新的登录提供商。

## 数据模型

### OAuth2 绑定
```go
type OAuth2Binding struct {
    ID        string `gorm:"primaryKey"`
    UserID    string
    Provider  string    // 提供商类型 (github, google, etc.)
    ProviderID string   // 第三方用户 ID
    Data      []byte    // 额外数据
    CreatedAt time.Time
}
```

### 厂商后端
```go
type VendorBackend struct {
    ID         string
    Name       string
    Type       string    // bilibili, alist, emby
    URL        string
    Token      string
    Username   string
    Password   string
    CreatedAt  time.Time
    UpdatedAt  time.Time
}
```

## OAuth2 流程

```
1. 前端请求授权 URL
   GET /api/oauth2/:provider/auth
   
2. 用户跳转到第三方登录页面
   
3. 第三方回调
   GET /api/oauth2/:provider/callback?code=xxx
   
4. 后端获取 access_token
   
5. 后端获取用户信息
   
6. 查找或创建 SyncTV 用户
   
7. 生成 JWT Token
   
8. 返回登录成功
```

## 配置示例

```yaml
oauth2_plugins:
  github:
    client_id: "your_client_id"
    client_secret: "your_client_secret"
    redirect_url: "http://your-domain.com/api/oauth2/github/callback"
    
  google:
    client_id: "your_client_id"
    client_secret: "your_client_secret"
    redirect_url: "http://your-domain.com/api/oauth2/google/callback"
```

## 依赖关系

| 依赖模块 | 用途 |
|----------|------|
| `golang.org/x/oauth2` | OAuth2 客户端 |
| `internal/model` | 数据模型 |
| `internal/db` | 数据库访问 |
| `proto/provider` | Protobuf 消息 |

## 重要注意事项

- 每个提供商需要在对应平台注册应用
- 回调 URL 必须完全匹配
- 支持 HTTP 和 HTTPS 回调
- 插件需要在启动时加载
