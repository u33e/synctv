# [根目录](../CLAUDE.md) > internal/model (数据模型层)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / model (数据模型层)

## 模块职责

`internal/model` 模块定义了 SyncTV 的所有数据模型，使用 GORM 进行数据库 ORM 映射。包含用户、房间、影片、成员、设置等核心实体。

## 目录结构

```
internal/model/
├── room.go            # 房间模型
├── movie.go           # 影片模型
├── user.go            # 用户模型
├── member.go          # 房间成员模型
├── current.go         # 当前播放状态
├── setting.go         # 设置模型
├── oauth2.go         # OAuth2 关联模型
├── vendorBackend.go   # 厂商后端模型
└── vendorRecord.go    # 厂商记录模型
```

## 数据模型概览

### User (用户)

```go
type User struct {
    ID                    string          // 用户 ID
    Username              string          // 用户名 (唯一)
    Email                 EmptyNullString // 邮箱 (可选)
    HashedPassword        []byte          // 密码哈希
    Role                  Role            // 用户角色
    RegisteredByProvider  bool            // 是否通过提供商注册
    RegisteredByEmail     bool            // 是否通过邮箱注册
    // 关联...
}
```

**角色 (Role)**:
- `RoleBanned` - 已封禁
- `RolePending` - 待审核
- `RoleUser` - 普通用户
- `RoleAdmin` - 管理员
- `RoleRoot` - 超级管理员

### Room (房间)

```go
type Room struct {
    ID             string        // 房间 ID
    Name           string        // 房间名称 (唯一)
    CreatorID      string        // 创建者 ID
    HashedPassword []byte        // 房间密码 (可选)
    Status         RoomStatus    // 房间状态
    Settings       *RoomSettings // 房间设置
    Current        *Current      // 当前播放状态
    // 关联...
}
```

**状态 (RoomStatus)**:
- `RoomStatusBanned` - 已封禁
- `RoomStatusPending` - 待审核
- `RoomStatusActive` - 活跃

### Movie (影片)

```go
type Movie struct {
    ID        string     // 影片 ID
    RoomID    string     // 所属房间 ID
    CreatorID string     // 创建者 ID
    Position  uint       // 位置 (排序)
    MovieBase MovieBase  // 影片基本信息
    // 关联...
}
```

### MovieBase (影片基本信息)

```go
type MovieBase struct {
    URL         string               // 影片 URL
    Name        string               // 影片名称
    Type        string               // 影片类型
    Headers     map[string]string    // 自定义请求头
    Subtitles   map[string]*Subtitle // 字幕
    MoreSources []*MoreSource       // 备用源
    Danmu       string               // 弹幕 URL
    Live        bool                 // 是否直播
    Proxy       bool                 // 是否代理
    IsFolder    bool                 // 是否文件夹
    VendorInfo  VendorInfo          // 厂商信息
}
```

### VendorInfo (厂商信息)

```go
type VendorInfo struct {
    Bilibili *BilibiliStreamingInfo // Bilibili 信息
    Alist    *AlistStreamingInfo    // Alist 信息
    Emby     *EmbyStreamingInfo     // Emby 信息
    Vendor   VendorName             // 厂商名称
    Backend  string                 // 后端名称
}
```

**支持的厂商**:
- `VendorBilibili` ("bilibili")
- `VendorAlist` ("alist")
- `VendorEmby` ("emby")

### RoomMember (房间成员)

```go
type RoomMember struct {
    RoomID        string             // 房间 ID
    UserID       string             // 用户 ID
    Role         RoomMemberRole      // 成员角色
    Status       RoomMemberStatus    // 成员状态
}
```

**成员角色**:
- `RoleGuest` - 访客
- `RoleUser` - 用户
- `RoleAdmin` - 管理员
- `RoleCreator` - 创建者

**成员状态**:
- `StatusPending` - 待审核
- `StatusActive` - 活跃
- `StatusBanned` - 已封禁

### Current (当前播放状态)

```go
type Current struct {
    RoomID       string    // 房间 ID
    MovieID      string    // 当前影片 ID
    IsPlaying    bool      // 是否播放中
    CurrentTime  float64   // 当前时间
    PlaybackRate float64   // 播放速度
}
```

### Setting (设置)

```go
type Setting struct {
    Name  string        // 设置名称 (唯一)
    Type  SettingType   // 设置类型
    Value string        // 设置值
    Group SettingGroup  // 设置分组
}
```

**设置类型**:
- `SettingTypeString`
- `SettingTypeInt64`
- `SettingTypeFloat64`
- `SettingTypeBool`

**设置分组**:
- `SettingGroupServer`
- `SettingGroupEmail`
- `SettingGroupOAuth2`

### VendorBackend (厂商后端)

```go
type VendorBackend struct {
    Backend model.Backend    // 后端配置
    UsedBy  model.UsedBy     // 使用配置
}
```

### UserProvider (用户提供商绑定)

```go
type UserProvider struct {
    UserID    string // 用户 ID
    Provider  string // 提供商名称
    ProviderID string // 提供商用户 ID
    Username  string // 提供商用户名
}
```

## 对外接口

该模块仅定义数据结构，通过 `internal/db` 模块提供数据库访问。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `gorm.io/gorm` | ORM 框架 |
| `golang.org/x/crypto/bcrypt` | 密码加密 |

## 数据库迁移

模型变更需要通过 GORM 的 `AutoMigrate` 处理:

```go
db.AutoMigrate(
    &model.User{},
    &model.Room{},
    &model.Movie{},
    &model.RoomMember{},
    // ...
)
```

## 常见问题 (FAQ)

### 如何添加新模型?

1. 在 `internal/model/` 中定义模型
2. 在 `internal/db/` 中添加对应的 CRUD 方法
3. 在 `internal/bootstrap/db.go` 中添加到 `AutoMigrate`

### 如何自定义字段序列化?

使用 GORM 的 serializer:

```go
type MyModel struct {
    MyMap map[string]string `gorm:"serializer:fastjson;type:text"`
}
```

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [room.go](room.go) | 房间模型 |
| [movie.go](movie.go) | 影片模型 |
| [user.go](user.go) | 用户模型 |
| [member.go](member.go) | 成员模型 |
| [current.go](current.go) | 播放状态 |
| [setting.go](setting.go) | 设置模型 |
| [oauth2.go](oauth2.go) | OAuth2 关联 |
| [vendorBackend.go](vendorBackend.go) | 厂商后端 |
| [vendorRecord.go](vendorRecord.go) | 厂商记录 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
