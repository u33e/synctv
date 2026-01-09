# [CLAUDE.md](../CLAUDE.md) > server 模块

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / server (HTTP 服务层)

## 模块概述

`server` 是 SyncTV 的 HTTP 服务层，基于 Gin 框架实现，负责处理所有 HTTP 请求、WebSocket 连接、第三方厂商集成和视频代理。

## 目录结构

```
server/
├── router.go                      # 路由初始化入口
├── handlers/                      # HTTP 处理器
│   ├── init.go                   # 路由注册入口
│   ├── admin.go                  # 管理员 API
│   ├── danmu.go                  # 弹幕 API
│   ├── member.go                 # 成员 API
│   ├── movie.go                  # 影片 API
│   ├── public.go                 # 公开 API
│   ├── room.go                   # 房间 API
│   ├── root.go                   # Root 用户 API
│   ├── user.go                   # 用户 API
│   ├── websocket.go              # WebSocket 处理器
│   ├── proxy/                    # 视频代理模块
│   │   ├── buffer.go            # 缓冲区管理
│   │   ├── cache.go             # 代理缓存
│   │   ├── m3u8.go              # M3U8 解析
│   │   ├── proxy.go             # HTTP 代理
│   │   ├── readseeker.go        # 流式读取
│   │   └── slice.go             # 分片处理
│   └── vendors/                  # 第三方厂商集成
│       ├── vendorbilibili/      # Bilibili 集成
│       ├── vendoralist/          # Alist 集成
│       └── vendoremby/          # Emby 集成
├── middlewares/                  # 中间件
│   ├── auth.go                  # 认证中间件
│   ├── cors.go                  # CORS 中间件
│   ├── init.go                  # 中间件初始化
│   ├── log.go                   # 日志中间件
│   └── rateLimit.go             # 限流中间件
├── oauth2/                      # OAuth2 处理
├── static/                      # 静态文件服务
└── model/                       # Server 数据模型
```

## 路由结构

### 公开 API (`/api/public/`)
- `GET /api/public/settings` - 获取公开设置

### 管理员 API (`/api/admin/`)
- `GET /api/admin/settings` - 获取设置
- `POST /api/admin/settings` - 修改设置
- `GET /api/admin/user/list` - 用户列表
- `POST /api/admin/user/add` - 添加用户
- `POST /api/admin/user/delete` - 删除用户
- `GET /api/admin/room/list` - 房间列表
- `POST /api/admin/room/delete` - 删除房间

### 用户 API (`/api/user/`)
- `POST /api/user/login` - 用户登录
- `POST /api/user/signup` - 用户注册
- `GET /api/user/me` - 获取用户信息
- `POST /api/user/logout` - 退出登录
- `GET /api/user/rooms` - 用户房间列表

### 房间 API (`/api/room/`)
- `GET /api/room/check` - 检查房间
- `GET /api/room/list` - 房间列表
- `POST /api/room/create` - 创建房间
- `GET /api/room/me` - 当前房间信息
- `GET /api/room/ws` - WebSocket 连接
- `GET /api/room/members` - 房间成员

### 影片 API (`/api/room/movie/`)
- `GET /api/room/movie/current` - 当前播放影片
- `GET /api/room/movie/movies` - 影片列表
- `POST /api/room/movie/push` - 添加影片
- `POST /api/room/movie/delete` - 删除影片
- `HEAD /api/room/movie/proxy/:movieId` - 视频代理
- `GET /api/room/movie/proxy/:movieId/m3u8/:targetToken` - M3U8 服务

### 厂商 API (`/api/vendor/`)
- `/api/vendor/bilibili/` - Bilibili 相关
- `/api/vendor/alist/` - Alist 相关
- `/api/vendor/emby/` - Emby 相关

## 核心组件

### 1. Proxy (视频代理)
**目录**: [handlers/proxy/](handlers/proxy/)

**职责**:
- HTTP 视频流代理
- M3U8 播放列表解析
- HLS 分片缓存
- 流式读取支持 (Seekable)

**关键文件**:
| 文件 | 功能 |
|------|------|
| [proxy.go](handlers/proxy/proxy.go) | HTTP 代理入口 |
| [cache.go](handlers/proxy/cache.go) | 分片缓存机制 |
| [m3u8.go](handlers/proxy/m3u8.go) | M3U8 解析器 |
| [slice.go](handlers/proxy/slice.go) | 分片处理 |
| [readseeker.go](handlers/proxy/readseeker.go) | io.ReadSeeker 实现 |

### 2. Middleware (中间件)
**目录**: [middlewares/](middlewares/)

**中间件列表**:
| 中间件 | 文件 | 功能 |
|--------|------|------|
| Auth | [auth.go](middlewares/auth.go) | 用户/管理员/房间认证 |
| CORS | [cors.go](middlewares/cors.go) | 跨域处理 |
| Log | [log.go](middlewares/log.go) | 请求日志 |
| RateLimit | [rateLimit.go](middlewares/rateLimit.go) | 速率限制 |

**认证中间件类型**:
- `AuthUserMiddleware` - 需要登录用户
- `AuthAdminMiddleware` - 需要管理员权限
- `AuthRootMiddleware` - 需要 Root 权限
- `AuthRoomMiddleware` - 需要房间成员
- `AuthRoomAdminMiddleware` - 需要房间管理员
- `AuthRoomCreatorMiddleware` - 需要房间创建者

### 3. WebSocket Handler
**文件**: [handlers/websocket.go](handlers/websocket.go)

**职责**:
- 处理 WebSocket 连接升级
- 消息路由到 Hub
- 处理客户端心跳
- 错误处理和重连

### 4. Vendors (厂商集成)
**目录**: [handlers/vendors/](handlers/vendors/)

**支持的厂商**:
- **Bilibili**: 直播/视频解析、扫码登录
- **Alist**: 文件列表浏览
- **Emby**: 媒体服务器集成

## 依赖关系

| 依赖模块 | 用途 |
|----------|------|
| `github.com/gin-gonic/gin` | Web 框架 |
| `internal/op` | 业务操作层 (Hub/Room/Movie/User) |
| `internal/model` | 数据模型 |
| `internal/bootstrap` | 初始化服务 |
| `cmd/flags` | 命令行参数 |

## 初始化流程

```
router.go (NewAndInit)
    ↓
middlewares.Init()      # 注册中间件
    ↓
oauth2.Init()          # 注册 OAuth2 路由
    ↓
handlers.Init()        # 注册 API 路由
    ↓
static.Init()          # 注册静态文件 (可选)
```

## 重要注意事项

- 所有需要认证的 API 都应使用相应的中间件
- WebSocket 连接需要先登录房间
- 视频代理支持 M3U8 和直接流媒体
- 厂商集成需要用户先绑定对应账号
