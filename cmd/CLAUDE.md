# [根目录](../CLAUDE.md) > cmd (CLI 命令层)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / cmd

## 模块职责

`cmd` 模块是 SyncTV 的命令行接口层，基于 [Cobra](https://github.com/spf13/cobra) 框架实现。负责处理用户命令、环境变量解析和程序启动。

## 目录结构

```
cmd/
├── root.go                # 根命令，环境变量处理
├── server.go              # Server 命令，HTTP/RTMP 服务启动
├── version.go             # 版本命令
├── self-update.go         # 自更新命令
├── flags/                # 命令行参数定义
│   ├── vars.go           # 全局变量
│   ├── server.go         # Server 相关参数
│   └── config.go        # 配置相关参数
├── admin/                # 管理员命令
│   ├── admin.go         # Admin 根命令
│   ├── add.go           # 添加用户
│   ├── delete.go        # 删除用户
│   └── show.go          # 显示用户
├── user/                 # 用户命令
│   ├── user.go          # User 根命令
│   ├── ban.go           # 封禁用户
│   ├── delete.go        # 删除用户
│   ├── search.go        # 搜索用户
│   └── unban.go        # 解封用户
├── root/                 # Root 用户命令
│   ├── root.go          # Root 根命令
│   ├── add.go           # 添加 root 用户
│   ├── delete.go        # 删除 root 用户
│   └── show.go          # 显示 root 用户
└── setting/              # 设置命令
    ├── setting.go       # Setting 根命令
    ├── set.go           # 设置值
    └── show.go          # 显示值
```

## 入口与启动

### 主入口: `main.go` (项目根目录)

```go
func main() {
    cmd.Execute()
}
```

### 根命令: `cmd/root.go`

负责:
- 加载 `.env` 文件
- 解析环境变量 (支持 `SYNCTV_` 前缀)
- 处理全局标志

### Server 命令: `cmd/server.go`

启动流程:
```
PreRunE (初始化)
    ↓
bootstrap.InitSysNotify    # 系统通知
    ↓
bootstrap.InitConfig      # 配置加载
    ↓
bootstrap.InitGinMode     # Gin 模式
    ↓
bootstrap.InitLog         # 日志初始化
    ↓
bootstrap.InitDatabase    # 数据库初始化
    ↓
bootstrap.InitProvider    # OAuth2 提供商
    ↓
bootstrap.InitOp         # 操作层初始化
    ↓
bootstrap.InitRtmp       # RTMP 初始化
    ↓
bootstrap.InitVendorBackend # 厂商后端
    ↓
bootstrap.InitSetting    # 设置系统
    ↓
Run (启动服务)
    ↓
HTTP Server + RTMP Server (通过 cmux 多路复用)
```

## 命令列表

| 命令 | 用途 | 示例 |
|------|------|------|
| `synctv server` | 启动服务器 | `synctv server --data-dir .` |
| `synctv admin add` | 添加管理员 | `synctv admin add username password` |
| `synctv user ban` | 封禁用户 | `synctv user ban userId` |
| `synctv setting set` | 设置配置 | `synctv setting set key value` |
| `synctv self-update` | 自更新 | `synctv self-update` |

## 环境变量

所有环境变量支持 `SYNCTV_` 前缀 (可通过 `--env-no-prefix` 禁用):

| 变量 | 说明 |
|------|------|
| `DATA_DIR` | 数据目录 |
| `DEV` | 开发模式 |
| `SKIP_CONFIG` | 跳过配置文件 |
| `DISABLE_WEB` | 禁用 Web 前端 |

## 命令行参数

### 全局参数 (`--global`)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--data-dir` | 数据目录 | `~/.synctv` |
| `--dev` | 开发模式 | `false` |
| `--log-std` | 日志输出到标准输出 | `true` |
| `--force-auto-migrate` | 强制自动迁移 | `dev 时为 true` |
| `--github-base-url` | GitHub API 地址 | `https://api.github.com/` |

### Server 参数

| 参数 | 说明 |
|------|------|
| `--disable-update-check` | 禁用更新检查 |
| `--disable-web` | 禁用 Web 前端 |
| `--disable-log-color` | 禁用日志颜色 |
| `--web-path` | Web 前端路径 |
| `--skip-config` | 跳过配置文件 |
| `--skip-env-config` | 跳过环境变量配置 |

## 对外接口

该模块不提供对外 API，仅作为 CLI 入口。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/spf13/cobra` | CLI 框架 |
| `github.com/joho/godotenv` | `.env` 文件加载 |
| `github.com/caarlos0/env/v9` | 环境变量解析 |
| `github.com/synctv-org/synctv/internal/bootstrap` | 启动初始化 |
| `github.com/synctv-org/synctv/internal/version` | 版本管理 |

## 配置文件

支持从数据目录加载 `.env` 文件:
- `.env`
- `.env.local`
- `.env.*`

## 常见问题 (FAQ)

### 如何指定不同的数据目录?

```bash
synctv server --data-dir /path/to/data
```

或通过环境变量:

```bash
export SYNCTV_DATA_DIR=/path/to/data
synctv server
```

### 如何禁用 Web 前端?

```bash
synctv server --disable-web
```

### 如何启用开发模式?

```bash
synctv server --dev
```

开发模式下会:
- 加载当前目录的 `.env` 文件
- 启用强制自动迁移
- 禁用更新检查 (如配置)

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [cmd/root.go](root.go) | 根命令实现 |
| [cmd/server.go](server.go) | Server 命令实现 |
| [cmd/version.go](version.go) | 版本命令 |
| [cmd/self-update.go](self-update.go) | 自更新命令 |
| [cmd/flags/vars.go](flags/vars.go) | 全局标志 |
| [cmd/admin/admin.go](admin/admin.go) | 管理员命令 |
| [cmd/user/user.go](user/user.go) | 用户命令 |
| [cmd/setting/setting.go](setting/setting.go) | 设置命令 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
