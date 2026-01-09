# [根目录](../CLAUDE.md) > internal/sysNotify (系统通知)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / sysNotify (系统通知)

## 模块职责

`internal/sysNotify` 模块提供系统信号监听和优雅关闭机制，支持在程序退出或重载时执行清理任务。

## 目录结构

```
internal/sysNotify/
├── sysnotify.go       # 系统通知核心
└── signal*.go         # 平台相关信号处理
```

## 通知类型

```go
type NotifyType int

const (
    NotifyTypeEXIT    NotifyType = iota + 1 // 退出信号
    NotifyTypeRELOAD                        // 重载信号
)
```

## 核心概念

### Task (任务)

```go
type Task struct {
    Task       func() error    // 任务函数
    Name       string          // 任务名称
    NotifyType NotifyType     // 关联的信号类型
}
```

### 任务队列

使用优先级队列管理任务，按优先级顺序执行。

## 对外接口

### 初始化

```go
func Init()
```

初始化系统信号监听。

### 注册任务

```go
func RegisterSysNotifyTask(priority int, task *Task) error
```

**参数**:
- `priority`: 任务优先级 (数值越大越先执行)
- `task`: 任务对象

### 等待通知

```go
func WaitCbk()
```

阻塞直到收到退出信号，执行所有已注册的任务后返回。

### 创建任务

```go
func NewSysNotifyTask(name string, notifyType NotifyType, task func() error) *Task
```

## 使用示例

### 注册退出任务

```go
err := sysnotify.RegisterSysNotifyTask(
    100, // 优先级
    sysnotify.NewSysNotifyTask(
        "close-database",
        sysnotify.NotifyTypeEXIT,
        func() error {
            return db.Close()
        },
    ),
)
```

### 注册重载任务

```go
err := sysnotify.RegisterSysNotifyTask(
    50,
    sysnotify.NewSysNotifyTask(
        "reload-config",
        sysnotify.NotifyTypeRELOAD,
        func() error {
            return config.Reload()
        },
    ),
)
```

### 在主程序中使用

```go
func main() {
    // 初始化
    sysnotify.Init()

    // 注册任务
    sysnotify.RegisterSysNotifyTask(100, closeDatabaseTask)
    sysnotify.RegisterSysNotifyTask(50, closeConnectionsTask)

    // 启动服务
    go startServer()

    // 等待退出信号
    sysnotify.WaitCbk()
}
```

## 执行流程

### 退出流程

```
收到退出信号 (SIGTERM/SIGINT)
    ↓
获取 NotifyTypeEXIT 任务队列
    ↓
按优先级执行任务 (高优先级先执行)
    ↓
任务执行完成
    ↓
返回
```

### 重载流程

```
收到重载信号 (SIGHUP)
    ↓
获取 NotifyTypeRELOAD 任务队列
    ↓
按优先级执行任务
    ↓
继续运行
```

## 信号处理

### Unix/Linux/Mac

- `SIGINT` (Ctrl+C) -> NotifyTypeEXIT
- `SIGTERM` -> NotifyTypeEXIT
- `SIGHUP` -> NotifyTypeRELOAD

### Windows

- 见 `signal_windows.go`

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/zijiren233/gencontainer/pqueue` | 优先级队列 |
| `github.com/zijiren233/gencontainer/rwmap` | 读写映射 |

## 常见问题 (FAQ)

### 如何设置任务执行顺序?

通过 `priority` 参数，数值越大越先执行。

### 任务执行失败会怎样?

单个任务失败不会影响其他任务执行，错误会记录到日志。

### 如何支持跨平台?

使用 `signal.go` 和 `signal_windows.go` 平台相关文件。

## 错误处理

- 任务函数返回的错误会被记录到日志
- 任务 panic 会被捕获并记录

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [sysnotify.go](sysnotify.go) | 系统通知核心 |
| [signal.go](signal.go) | Unix/Linux/Mac 信号处理 |
| [signal_windows.go](signal_windows.go) | Windows 信号处理 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
