# [CLAUDE.md](../../CLAUDE.md) > internal/rtmp 模块

> [!NOTE]
> 面包屑: [SyncTV 项目根](../../CLAUDE.md) / [internal](../) / rtmp (RTMP 服务)

## 模块概述

`internal/rtmp` 负责处理 RTMP 流媒体服务，支持实时直播推流和拉流。

## 目录结构

```
internal/rtmp/
└── rtmp.go          # RTMP 服务主文件
```

## 核心功能

### RTMP 推流认证

使用 JWT 进行推流认证，确保只有授权用户可以推流。

```go
// 认证推流请求
func AuthRtmpPublish(authorization string) (movieID string, err error)

// 生成推流授权 Token
func NewRtmpAuthorization(movieID string) (string, error)
```

### JWT Claims

```go
type Claims struct {
    MovieID string `json:"m"`       // 影片/房间 ID
    jwt.RegisteredClaims
}
```

## RTMP 流程

```
1. 用户申请推流密钥
   → NewRtmpAuthorization(movieID) 返回 JWT Token
   
2. 用户使用推流密钥推流
   rtmp://server/live/stream?token=xxx
   
3. 服务端验证 Token
   → AuthRtmpPublish() 验证 JWT 并提取 MovieID
   
4. 推流成功，其他用户可观看
```

## 端口复用

RTMP 可与 HTTP 共用端口 (通过 cmux):
- 相同地址和端口时启用多路复用
- 自动识别 HTTP 和 RTMP 流量

## 依赖关系

| 依赖模块 | 用途 |
|----------|------|
| `github.com/zijiren233/livelib/server` | RTMP 服务器 |
| `github.com/golang-jwt/jwt/v5` | JWT 认证 |
| `internal/conf` | 配置访问 |

## 配置

```yaml
server:
  rtmp:
    enable: true
    listen: "0.0.0.0"
    port: 1935
```

## 重要注意事项

- 推流必须携带有效的 JWT Token
- Token 中包含 MovieID，用于关联房间
- 支持 RTMP 直播流的 HLS 转码
- HTTP 和 RTMP 可共用端口
