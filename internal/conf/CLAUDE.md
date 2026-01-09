# [根目录](../CLAUDE.md) > internal/conf (配置系统)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / conf (配置系统)

## 模块职责

`internal/conf` 模块定义 SyncTV 的配置结构，负责从配置文件和环境变量加载配置。

## 目录结构

```
internal/conf/
├── config.go     # 主配置结构
├── db.go        # 数据库配置
├── jwt.go       # JWT 配置
├── log.go       # 日志配置
├── oauth2.go    # OAuth2 配置
├── reatLimit.go # 速率限制配置
├── server.go    # 服务器配置
└── var.go       # 配置变量
```

## 配置结构

### Config (主配置)

```go
type Config struct {
    Log         LogConfig         // 日志配置
    Server      ServerConfig      // 服务器配置
    Jwt         JwtConfig         // JWT 配置
    Database    DatabaseConfig    // 数据库配置
    Oauth2Plugins Oauth2Plugins  // OAuth2 插件配置
    RateLimit   RateLimitConfig   // 速率限制配置
}
```

### LogConfig (日志配置)

```go
type LogConfig struct {
    Level     string // 日志级别
    FilePath  string // 日志文件路径
    Color     bool   // 是否启用颜色
}
```

### ServerConfig (服务器配置)

```go
type ServerConfig struct {
    HTTP struct {
        Listen    string // 监听地址
        Port      uint16 // 监听端口
        CertPath  string // TLS 证书路径
        KeyPath   string // TLS 密钥路径
    }
    RTMP struct {
        Enable bool   // 是否启用 RTMP
        Listen string // 监听地址
        Port   uint16 // 监听端口
    }
    ProxyCachePath string // 代理缓存路径
}
```

### JwtConfig (JWT 配置)

```go
type JwtConfig struct {
    Secret string // JWT 密钥
    Expire string // 过期时间 (如 "24h")
}
```

### DatabaseConfig (数据库配置)

```go
type DatabaseType string

const (
    DatabaseTypeSqlite3   DatabaseType = "sqlite3"
    DatabaseTypeMysql     DatabaseType = "mysql"
    DatabaseTypePostgres  DatabaseType = "postgres"
)

type DatabaseConfig struct {
    Type     DatabaseType // 数据库类型
    Name     string      // 数据库名称
    Host     string      // 主机地址
    Port     uint16     // 端口
    User     string      // 用户名
    Password string      // 密码
}
```

### Oauth2Plugins (OAuth2 插件配置)

```go
type Oauth2Plugins struct {
    Plugins  map[string]ProviderPlugin // 插件列表
    Default  string                    // 默认提供商
    Redirect string                    // 重定向 URL
}

type ProviderPlugin struct {
    ClientID     string // 客户端 ID
    ClientSecret string // 客户端密钥
    RedirectURL  string // 重定向 URL
}
```

### RateLimitConfig (速率限制配置)

```go
type RateLimitConfig struct {
    Enabled bool     // 是否启用
    Rate    string   // 速率 (如 "10/s")
    Burst   int      // 突发容量
}
```

## 默认配置

```go
func DefaultConfig() *Config
```

**默认值**:
- 日志级别: `info`
- HTTP 监听: `0.0.0.0:8080`
- 数据库类型: `sqlite3`
- JWT 过期: `24h`

## 配置加载

配置从以下来源加载 (优先级从高到低):

1. 环境变量 (`SYNCTV_` 前缀)
2. 配置文件 (YAML)
3. 默认值

### 环境变量映射

| 环境变量 | 配置路径 |
|----------|----------|
| `SYNCTV_LOG_LEVEL` | `log.level` |
| `SYNCTV_DATA_DIR` | 数据目录 |
| `SYNCTV_DB_TYPE` | `database.type` |
| `SYNCTV_SERVER_HTTP_PORT` | `server.http.port` |
| `SYNCTV_JWT_SECRET` | `jwt.secret` |

## 对外接口

该模块仅提供配置结构定义，不提供对外 API。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/synctv-org/synctv/utils` | YAML 读写 |

## 配置文件示例

```yaml
log:
  level: info
  file_path: ""
  color: true

server:
  http:
    listen: "0.0.0.0"
    port: 8080
    cert_path: ""
    key_path: ""
  rtmp:
    enable: false
    listen: "0.0.0.0"
    port: 8080
  proxy_cache_path: "/tmp/synctv"

jwt:
  secret: ""
  expire: 24h

database:
  type: "sqlite3"
  name: "synctv"
  host: "localhost"
  port: 3306
  user: ""
  password: ""
```

## 常见问题 (FAQ)

### 如何切换数据库类型?

通过环境变量:

```bash
export SYNCTV_DB_TYPE=mysql
export SYNCTV_DB_HOST=localhost
export SYNCTV_DB_PORT=3306
export SYNCTV_DB_USER=root
export SYNCTV_DB_PASSWORD=password
export SYNCTV_DB_NAME=synctv
```

### 如何修改日志级别?

```bash
export SYNCTV_LOG_LEVEL=debug
```

可用级别: `trace`, `debug`, `info`, `warn`, `error`, `fatal`, `panic`

### 如何启用 TLS?

```yaml
server:
  http:
    cert_path: "/path/to/cert.pem"
    key_path: "/path/to/key.pem"
```

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [config.go](config.go) | 主配置结构 |
| [db.go](db.go) | 数据库配置 |
| [jwt.go](jwt.go) | JWT 配置 |
| [log.go](log.go) | 日志配置 |
| [oauth2.go](oauth2.go) | OAuth2 配置 |
| [reatLimit.go](reatLimit.go) | 速率限制配置 |
| [server.go](server.go) | 服务器配置 |
| [var.go](var.go) | 配置变量 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
