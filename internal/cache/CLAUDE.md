# [根目录](../CLAUDE.md) > internal/cache (缓存系统)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / cache (缓存系统)

## 模块职责

`internal/cache` 模块为第三方厂商 (Bilibili, Alist, Emby) 提供缓存功能，减少 API 调用，提高响应速度。

## 目录结构

```
internal/cache/
├── cache.go      # 缓存核心
├── cache0.go     # 缓存实现
├── alist.go      # Alist 缓存
├── bilibili.go   # Bilibili 缓存
└── emby.go       # Emby 缓存
```

## 缓存类型

### Cache 接口

```go
type Cache[K comparable, V any] interface {
    Set(key K, value V, duration time.Duration)
    Get(key K) (V, bool)
    Delete(key K)
    Clear()
}
```

## 厂商缓存

### Bilibili 缓存

**文件**: [bilibili.go](bilibili.go)

**缓存内容**:
- 用户信息
- 视频信息
- 直播流信息

**使用示例**:

```go
cache.SetBilibiliUser(key, user, time.Hour*24)
user, found := cache.GetBilibiliUser(key)
```

### Alist 缓存

**文件**: [alist.go](alist.go)

**缓存内容**:
- 文件列表
- 文件元数据

**使用示例**:

```go
cache.SetAlistFileList(key, files, time.Minute*5)
files, found := cache.GetAlistFileList(key)
```

### Emby 缓存

**文件**: [emby.go](emby.go)

**缓存内容**:
- 媒体库信息
- 用户信息

**使用示例**:

```go
cache.SetEmbyLibrary(key, library, time.Hour)
library, found := cache.GetEmbyLibrary(key)
```

## 对外接口

该模块为内部使用，不提供直接的对外 API。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/zijiren233/gencontainer/synccache` | 同步缓存 |

## 使用示例

### 创建缓存

```go
c := cache0.NewCache[string, MyType]()

c.Set("key", value, time.Minute*10)
value, found := c.Get("key")
```

### 清除缓存

```go
c.Delete("key")
c.Clear()
```

## 常见问题 (FAQ)

### 缓存多久过期?

默认过期时间:
- Bilibili 用户信息: 24 小时
- Alist 文件列表: 5 分钟
- Emby 媒体库: 1 小时

### 缓存存储在哪里?

使用内存缓存，重启后缓存失效。

### 如何刷新缓存?

调用对应厂商的刷新方法，会自动更新缓存。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [cache.go](cache.go) | 缓存核心 |
| [cache0.go](cache0.go) | 缓存实现 |
| [alist.go](alist.go) | Alist 缓存 |
| [bilibili.go](bilibili.go) | Bilibili 缓存 |
| [emby.go](emby.go) | Emby 缓存 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
