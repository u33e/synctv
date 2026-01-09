# [根目录](../CLAUDE.md) > internal/version (版本管理)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / version (版本管理)

## 模块职责

`internal/version` 模块提供版本管理和自更新功能，从 GitHub 检查更新并支持自动下载安装新版本。

## 目录结构

```
internal/version/
├── version.go       # 版本信息和管理
└── update.go       # 自更新功能
```

## 版本信息

### 当前版本

```go
var (
    Version   = "dev"
    GitCommit string
)
```

编译时通过 `-ldflags` 注入版本信息:

```bash
go build -ldflags "-X github.com/synctv-org/synctv/internal/version.Version=v1.0.0"
```

### 版本比较

```go
const (
    VersionEqual = iota
    VersionGreater
    VersionLess
)

func CompVersion(v1, v2 string) (int, error)
```

支持语义化版本比较，包括预发布版本 (alpha, beta, rc)。

**比较规则**:
- 相同版本返回 `VersionEqual`
- 当前版本大于目标返回 `VersionGreater`
- 当前版本小于目标返回 `VersionLess`

**预发布版本优先级**: `alpha` < `beta` < `rc` < 正式版本

## 自更新功能

### Info 结构

```go
type Info struct {
    current string
    latest  *github.RepositoryRelease
    dev     *github.RepositoryRelease
    c       *github.Client
    baseURL string
}
```

### 创建版本信息

```go
func NewVersionInfo(conf ...InfoConf) (*Info, error)
```

**选项**:
```go
func WithBaseURL(baseURL string) InfoConf
```

用于自定义 GitHub Enterprise API 地址。

### 检查更新

```go
func (v *Info) CheckLatest(ctx context.Context) (string, error)
```

返回最新版本的 tag。

### 判断是否需要更新

```go
func (v *Info) NeedUpdate(ctx context.Context) (bool, error)
```

规则:
- `dev` 版本总是返回 `false`
- 比较当前版本与最新版本

### 获取二进制 URL

```go
func (v *Info) LatestBinaryURL(ctx context.Context) (string, error)
func (v *Info) DevBinaryURL(ctx context.Context) (string, error)
```

自动匹配当前操作系统和架构。

### 自更新

```go
func (v *Info) SelfUpdate(ctx context.Context) error
```

**流程**:
1. 判断是否需要更新
2. 获取最新二进制 URL
3. 下载并替换当前可执行文件

## 对外接口

该模块通过 CLI 命令 (`cmd/self-update.go`) 暴露自更新功能。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/google/go-github/v56/github` | GitHub API |
| `github.com/synctv-org/synctv/internal/model` | 模型定义 |
| `github.com/synctv-org/synctv/internal/settings` | 设置系统 |
| `github.com/synctv-org/synctv/utils` | 版本比较 |

## 使用示例

### 检查更新

```go
v, err := version.NewVersionInfo()
if err != nil {
    log.Errorf("new version info error: %v", err)
    return
}

latest, err := v.Latest(context.Background())
if err != nil {
    log.Errorf("check latest error: %v", err)
    return
}

log.Infof("latest version: %s", latest)
```

### 判断是否需要更新

```go
need, err := v.NeedUpdate(context.Background())
if err != nil {
    log.Errorf("check need update error: %v", err)
    return
}

if need {
    log.Info("need update")
}
```

### 执行自更新

```go
err := v.SelfUpdate(context.Background())
if err != nil {
    log.Errorf("self update error: %v", err)
    return
}

log.Info("update success")
```

## 常见问题 (FAQ)

### 如何自定义 GitHub API 地址?

```go
v, err := version.NewVersionInfo(
    version.WithBaseURL("https://github.enterprise.com/api/v3/"),
)
```

### dev 版本如何更新?

```bash
synctv self-update --dev
```

或设置环境变量:

```bash
export SYNCTV_DEV=true
synctv self-update
```

### 如何获取当前版本?

```bash
synctv --version
```

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [version.go](version.go) | 版本信息和管理 |
| [update.go](update.go) | 自更新功能 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
