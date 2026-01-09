# [根目录](../CLAUDE.md) > internal/vendor (厂商后端集成)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / vendor (厂商后端集成)

## 模块职责

`internal/vendor` 模块管理第三方厂商后端的 gRPC 连接，提供 Bilibili、Alist、Emby 等服务的客户端。支持动态添加/删除后端，并支持服务发现 (Consul/etcd)。

## 目录结构

```
internal/vendor/
├── vendor.go      # 厂商后端核心管理
├── bilibili.go    # Bilibili 客户端接口
├── alist.go       # Alist 客户端接口
└── emby.go        # Emby 客户端接口
```

## 核心概念

### BackendConn (后端连接)

```go
type BackendConn struct {
    Conn *grpc.ClientConn    // gRPC 连接
    Info *model.VendorBackend // 后端配置
}
```

### Clients (厂商客户端集合)

```go
type Clients struct {
    bilibili map[string]BilibiliInterface
    alist    map[string]AlistInterface
    emby     map[string]EmbyInterface
}
```

## 对外接口

### 初始化

```go
func Init(ctx context.Context) error
```

从数据库加载所有已配置的厂商后端并建立连接。

### 后端管理

**添加后端**:

```go
func AddVendorBackend(ctx context.Context, backend *model.VendorBackend) error
```

**删除后端**:

```go
func DeleteVendorBackend(_ context.Context, endpoint string) error
```

**更新后端**:

```go
func UpdateVendorBackend(ctx context.Context, backend *model.VendorBackend) error
```

**启用/禁用后端**:

```go
func EnableVendorBackend(_ context.Context, endpoint string) error
func DisableVendorBackend(_ context.Context, endpoint string) error
```

### 批量操作

```go
func EnableVendorBackends(_ context.Context, endpoints []string) error
func DisableVendorBackends(_ context.Context, endpoints []string) error
func DeleteVendorBackends(_ context.Context, endpoints []string) error
```

### 获取客户端

```go
func LoadClients() *Clients
func LoadConns() map[string]*BackendConn
```

## gRPC 连接配置

### NewGrpcConn

```go
func NewGrpcConn(ctx context.Context, conf *model.Backend) (*grpc.ClientConn, error)
```

**支持的连接方式**:

1. **直接连接**:
   - 指定端点地址: `host:port`
   - 默认端口: 80 (HTTP) / 443 (HTTPS)

2. **Consul 服务发现**:
   ```go
   conf.Consul.ServiceName = "my-service"
   ```
   通过 Consul 自动发现服务实例。

3. **etcd 服务发现**:
   ```go
   conf.Etcd.ServiceName = "my-service"
   ```
   通过 etcd 自动发现服务实例。

### 中间件

- **熔断器** (Circuit Breaker): 使用 SRE 算法
- **JWT 认证**: 支持 JWT token 签名验证

### TLS 支持

- 支持 TLS 安全连接
- 支持自定义 CA 证书
- 最低 TLS 版本: 1.2

## 厂商客户端接口

### BilibiliInterface

Bilibili 直播/视频服务客户端。

### AlistInterface

Alist 文件服务客户端。

### EmbyInterface

Emby 媒体服务器客户端。

## HTTP 客户端

```go
func NewHTTPClientConn(ctx context.Context, conf *model.Backend) (*http.Client, error)
```

与 gRPC 连接类似，支持直接连接、Consul、etcd 服务发现。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/go-kratos/kratos/v2` | Kratos 微服务框架 |
| `github.com/hashicorp/consul/api` | Consul 服务发现 |
| `go.etcd.io/etcd/client/v3` | etcd 服务发现 |
| `github.com/synctv-org/synctv/internal/model` | 数据模型 |
| `github.com/synctv-org/synctv/internal/db` | 数据库访问 |

## 使用示例

### 添加厂商后端

```go
backend := &model.VendorBackend{
    Backend: model.Backend{
        Endpoint: "vendors.example.com:443",
        TLS:      true,
    },
    UsedBy: model.UsedBy{
        Enabled:             true,
        Bilibili:           true,
        BilibiliBackendName: "bilibili",
    },
}

err := vendor.AddVendorBackend(ctx, backend)
```

### 使用厂商客户端

```go
clients := vendor.LoadClients()
if bilibiliClient, ok := clients.BilibiliClients()["bilibili"]; ok {
    resp, err := bilibiliClient.GetVideo(ctx, req)
}
```

## 常见问题 (FAQ)

### 如何启用多个后端?

添加多个后端配置，每个后端可以使用不同的服务发现方式。

### 后端连接失败会怎样?

熔断器会暂时停止请求，防止雪崩。可以通过配置调整熔断参数。

### 如何切换后端?

```go
// 禁用旧后端
vendor.DisableVendorBackend(ctx, "old-endpoint:443")

// 启用新后端
vendor.EnableVendorBackend(ctx, "new-endpoint:443")
```

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [vendor.go](vendor.go) | 厂商后端核心管理 |
| [bilibili.go](bilibili.go) | Bilibili 客户端接口 |
| [alist.go](alist.go) | Alist 客户端接口 |
| [emby.go](emby.go) | Emby 客户端接口 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
