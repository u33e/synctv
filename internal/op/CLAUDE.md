# [CLAUDE.md](../../CLAUDE.md) > internal/op 模块

> [!NOTE]
> 面包屑: [SyncTV 项目根](../../CLAUDE.md) / [internal](../) / op (业务操作层)

## 模块概述

`internal/op` 是 SyncTV 的核心业务操作层，负责处理房间、用户、影片、Hub (WebSocket) 等核心业务逻辑。

## 目录结构

```
internal/op/
├── client.go      # WebSocket 客户端连接
├── current.go     # 当前播放状态
├── hub.go         # WebSocket Hub (消息广播中心)
├── message.go     # 消息定义
├── movies.go      # 影片列表批量操作
├── movie.go       # 单个影片操作
├── op.go          # 操作层初始化
├── room.go        # 房间管理核心逻辑
├── rooms.go       # 房间列表/搜索
├── user.go        # 用户管理核心逻辑
└── users.go       # 用户列表/搜索
```

## 核心组件

### 1. Hub (WebSocket Hub)
**文件**: [hub.go](hub.go)

**职责**:
- 管理所有 WebSocket 连接
- 广播消息到房间/用户
- 处理连接注册/注销
- 心跳检测 (5秒间隔)
- 在线人数统计

**关键类型**:
```go
type Hub struct {
    broadcast chan *broadcastMessage  // 广播消息通道
    exit      chan struct{}          // 退出信号
    id        string                  // Hub 标识符
    clients   rwmap.RWMap[string, *clients]  // 按 userID 索引的客户端
    wg        sync.WaitGroup
    once      utils.Once
    closed    uint32
}
```

**关键方法**:
| 方法 | 说明 |
|------|------|
| `Broadcast(data, ...conf)` | 广播消息到所有客户端 |
| `RegClient(cli)` | 注册新的 WebSocket 连接 |
| `UnRegClient(cli)` | 注销 WebSocket 连接 |
| `SendToUser(userID, data)` | 发送消息到指定用户 |
| `SendToConnID(userID, connID, data)` | 发送消息到指定连接 |
| `IsOnline(userID)` | 检查用户是否在线 |
| `ClientNum()` | 获取当前在线客户端数 |

**广播配置选项**:
- `WithIgnoreConnID(...)` - 忽略指定连接 ID
- `WithIgnoreID(...)` - 忽略指定用户 ID
- `WithRTCJoined()` - 仅发送给已加入 WebRTC 的用户

### 2. Client (WebSocket 客户端)
**文件**: [client.go](client.go)

**职责**:
- 封装单个 WebSocket 连接
- 处理消息发送/接收
- 管理 WebRTC 状态
- 用户关联

### 3. Room Manager
**文件**: [room.go](room.go)

**职责**:
- 房间创建/删除
- 影片列表管理
- 成员权限管理 (成员/管理员/创建者)
- 房间设置 (密码、观看权限等)
- 同步状态维护

### 4. Movie Manager
**文件**: [movie.go](movie.go), [movies.go](movies.go)

**职责**:
- 影片添加/删除/编辑
- 影片播放控制
- 当前播放状态管理
- 影片排序

### 5. User Manager
**文件**: [user.go](user.go), [users.go](users.go)

**职责**:
- 用户注册/登录
- 用户资料管理
- 绑定第三方账号 (OAuth2)
- 用户权限管理

### 6. Current State
**文件**: [current.go](current.go)

**职责**:
- 管理房间当前播放状态
- 进度同步
- 播放控制 (播放/暂停/seek)

## 数据流

```
HTTP/WebSocket 请求
    ↓
handlers/ 处理器
    ↓
op/ 业务操作层 (Hub/Room/Movie/User)
    ↓
db/ 数据库持久化
    ↓
返回响应 / 广播更新
```

## 依赖关系

| 依赖模块 | 用途 |
|----------|------|
| `internal/db` | 数据库访问 |
| `internal/model` | 数据模型 |
| `proto/message` | Protobuf 消息定义 |
| `github.com/gorilla/websocket` | WebSocket 支持 |
| `github.com/zijiren233/gencontainer/rwmap` | 并发安全的 Map |

## 重要注意事项

- Hub 的广播是异步的，消息通过 channel 传递
- 客户端断开时会自动从 Hub 中注销
- 房间权限是分级的：成员 < 管理员 < 创建者
- 支持游客模式 (Guest)
