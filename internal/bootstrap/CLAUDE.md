# [CLAUDE.md](../../CLAUDE.md) > internal/bootstrap 模块

> [!NOTE]
> 面包屑: [SyncTV 项目根](../../CLAUDE.md) / [internal](../) / bootstrap (启动初始化)

## 模块概述

`internal/bootstrap` 负责服务器启动时的初始化流程编排，按顺序初始化各种组件和服务。

## 目录结构

```
internal/bootstrap/
├── config.go        # 配置文件加载
├── db.go            # 数据库初始化
├── gin.go           # Gin 模式设置
├── init.go          # 启动流程定义
├── log.go           # 日志初始化
├── op.go            # 操作层初始化
├── provider.go      # OAuth2 提供商初始化
├── rtmp.go          # RTMP 服务初始化
├── setting.go       # 动态设置初始化
├── sqlite.go        # SQLite 初始化
├── sqlite_cgo.go    # SQLite CGO 初始化
├── sysNotify.go     # 系统通知初始化
├── update.go        # 更新检查
└── vendor.go        # 厂商后端初始化
```

## 初始化顺序

```go
boot := bootstrap.New().Add(
    bootstrap.InitSysNotify,      // 1. 系统通知
    bootstrap.InitConfig,        // 2. 配置加载
    bootstrap.InitGinMode,      // 3. Gin 模式
    bootstrap.InitLog,           // 4. 日志
    bootstrap.InitDatabase,      // 5. 数据库
    bootstrap.InitProvider,       // 6. OAuth2 提供商
    bootstrap.InitOp,            // 7. 操作层
    bootstrap.InitRtmp,          // 8. RTMP
    bootstrap.InitVendorBackend,  // 9. 厂商后端
    bootstrap.InitSetting,       // 10. 动态设置
    bootstrap.InitCheckUpdate,   // 11. 更新检查 (可选)
)
```

## 核心组件

### 1. Config (配置加载)
**文件**: [config.go](config.go)

**职责**:
- 加载 YAML 配置文件
- 加载环境变量 (前缀 SYNCTV_)
- 合并配置并验证

**配置源优先级**:
1. 环境变量 (最高)
2. 配置文件
3. 默认值

### 2. Database (数据库初始化)
**文件**: [db.go](db.go)

**职责**:
- 初始化 GORM
- 支持数据库: SQLite3, MySQL, PostgreSQL
- 自动迁移数据表
- 创建默认 Root 用户

### 3. Log (日志初始化)
**文件**: [log.go](log.go)

**职责**:
- 配置 logrus
- 设置日志级别
- 日志文件输出 (可选)

### 4. Provider (OAuth2 提供商)
**文件**: [provider.go](provider.go)

**职责**:
- 初始化 OAuth2 提供商
- 加载配置文件中的提供商
- 支持插件模式

### 5. Op (操作层初始化)
**文件**: [op.go](op.go)

**职责**:
- 初始化 Hub
- 初始化 Room Manager
- 初始化 User Manager
- 初始化全局状态

### 6. RTMP (RTMP 服务)
**文件**: [rtmp.go](rtmp.go)

**职责**:
- 初始化 RTMP 服务器
- 配置监听地址
- JWT 认证设置

### 7. Setting (动态设置)
**文件**: [setting.go](setting.go)

**职责**:
- 初始化动态设置系统
- 从数据库加载设置
- 注册设置处理器

### 8. Update (更新检查)
**文件**: [update.go](update.go)

**职责**:
- 检查 GitHub Releases
- 对比版本号
- 通知有新版本

## 依赖关系

| 依赖模块 | 用途 |
|----------|------|
| `internal/conf` | 配置结构定义 |
| `internal/db` | 数据库访问 |
| `internal/op` | 业务操作层 |
| `internal/provider` | OAuth2 提供商 |
| `internal/rtmp` | RTMP 服务 |
| `internal/settings` | 动态设置 |
| `github.com/sirupsen/logrus` | 日志库 |
| `gorm.io/gorm` | ORM 框架 |

## 重要注意事项

- 初始化顺序很重要，必须按依赖关系执行
- 数据库初始化时会自动创建 Root 用户 (root/root)
- 环境变量前缀为 `SYNCTV_`
- 可通过 `--skip-config` 跳过配置文件加载
- 可通过 `--skip-env-config` 跳过环境变量加载
