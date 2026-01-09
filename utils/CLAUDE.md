# [根目录](../CLAUDE.md) > utils (工具库)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / utils (工具库)

## 模块职责

`utils` 模块提供通用工具函数，包括字符串处理、文件操作、UUID 生成、版本比较、WebSocket 等。

## 目录结构

```
utils/
├── utils.go                # 通用工具函数
├── crypto.go               # 加密工具
├── websocket.go            # WebSocket 工具
├── smtp/                  # SMTP 工具
│   ├── format.go          # 格式化
│   └── smtpool.go        # SMTP 连接池
├── m3u8/                 # M3U8 工具
│   └── m3u8.go           # M3U8 解析
└── fastJSONSerializer/     # 快速 JSON 序列化器
    └── fastJSONSerializer.go
```

## 通用工具

### 字符串处理

```go
// 随机字符串
func RandString(n int) string

// 随机字节
func RandBytes(n int) []byte

// LIKE 模糊匹配
func LIKE(s string) string
```

### UUID 处理

```go
// 生成有序 UUID (可排序)
func SortUUID() string

// 基于现有 UUID 生成有序 UUID
func SortUUIDWithUUID(src uuid.UUID) string
```

### 版本比较

```go
const (
    VersionEqual = iota
    VersionGreater
    VersionLess
)

// 比较版本
func CompVersion(v1, v2 string) (int, error)

// 拆分版本号
func SplitVersion(v string) ([]int, error)
```

支持语义化版本，包括预发布版本 (alpha, beta, rc)。

### 分页处理

```go
// 获取分页项
func GetPageItems[T any](items []T, page, pageSize int) []T

// 获取分页范围
func GetPageItemsRange(total, page, pageSize int) (start, end int)
```

### 集合操作

```go
// 查找索引
func Index[T comparable](items []T, item T) int

// 是否存在
func In[T comparable](items []T, item T) bool
```

### 文件操作

```go
// 文件是否存在
func Exists(name string) bool

// 写入 YAML 文件
func WriteYaml(file string, module any) error

// 读取 YAML 文件
func ReadYaml(file string, module any) error

// 优化文件路径 (相对转绝对)
func OptFilePath(filePath string) (string, error)
```

### URL/文件处理

```go
// 获取文件扩展名
func GetFileExtension(f string) string

// 获取 URL 扩展名
func GetURLExtension(u string) string

// 判断是否为 M3U8 URL
func IsM3u8Url(u string) bool
```

### HTTP 操作

```go
// Cookie 转换
func HTTPCookieToMap(c []*http.Cookie) map[string]string
func MapToHTTPCookie(m map[string]string) []*http.Cookie

// 禁止重定向的 HTTP 客户端
func NoRedirectHTTPClient() *http.Client

// 用户代理
const UA = `Mozilla/5.0 ...`
```

### 单次执行

```go
type Once struct {
    done uint32
    m    sync.Mutex
}

func (o *Once) Done() (doned bool)
func (o *Once) Do(f func())
func (o *Once) Reset()
```

### IP 地址处理

```go
// 解析 URL 是否为本地 IP
func ParseURLIsLocalIP(u string) (bool, error)

// 判断是否为本地 IP
func IsLocalIP(address string) bool
```

### 日志颜色

```go
// 判断是否需要颜色
func ForceColor() bool
```

### 环境变量处理

```go
// 获取 .env 文件列表
func GetEnvFiles(root string) ([]string, error)
```

### HTTP 请求处理

```go
// 从 Gin 上下文获取分页参数
func GetPageAndMax(ctx *gin.Context) (page, _max int, err error)
```

### 字符串截断

```go
// 按 Rune 长度截断
func TruncateByRune(s string, length int) string
```

## 加密工具

### Crypto (crypto.go)

```go
// 加密到 Base64
func CryptoToBase64(data, key []byte) (string, error)

// 从 Base64 解密
func DecryptoFromBase64(data string, key []byte) ([]byte, error)

// 生成加密密钥
func GenCryptoKey(password string) []byte
```

## WebSocket 工具

### WebSocket (websocket.go)

提供 WebSocket 消息处理相关工具。

## SMTP 工具

### SMTP 连接池 (smtp/smtpool.go)

管理 SMTP 连接池，复用连接提高性能。

### 格式化 (smtp/format.go)

邮件格式化相关工具。

## M3U8 工具

### M3U8 (m3u8/m3u8.go)

M3U8 播放列表解析和处理。

## 快速 JSON 序列化器

### FastJSONSerializer (fastJSONSerializer/fastJSONSerializer.go)

为 GORM 提供 fastjson 序列化器，提高 JSON 字段的读写性能。

## 常用常量

```go
// 用户代理
const UA = `Mozilla/5.0 ...`

// 版本比较结果
const (
    VersionEqual = iota
    VersionGreater
    VersionLess
)
```

## 对外接口

该模块不提供对外 API，供内部其他模块使用。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/gin-gonic/gin` | Web 框架 (HTTP 处理) |
| `github.com/google/uuid` | UUID 生成 |
| `gopkg.in/yaml.v3` | YAML 读写 |
| `github.com/zijiren233/yaml-comment` | 带注释的 YAML |

## 使用示例

### 生成随机字符串

```go
code := utils.RandString(6) // 6 位随机字符串
```

### 版本比较

```go
comp, err := utils.CompVersion("v1.0.0", "v1.1.0")
if err != nil {
    return err
}
if comp == utils.VersionLess {
    log.Info("current version is older")
}
```

### 文件操作

```go
// 写入配置
err := utils.WriteYaml("config.yaml", config)

// 读取配置
err := utils.ReadYaml("config.yaml", &config)
```

### 分页处理

```go
items := []string{"a", "b", "c", "d", "e"}
pageItems := utils.GetPageItems(items, 2, 2) // 第 2 页，每页 2 个
// 结果: ["c", "d"]
```

## 常见问题 (FAQ)

### SortUUID 与普通 UUID 的区别?

`SortUUID` 生成的 UUID 可以按字典序排序，适合用作数据库主键。

### 如何处理版本预发布标识?

`CompVersion` 自动识别 `alpha`, `beta`, `rc` 并正确比较。

### LIKE 函数的用途?

用于数据库模糊查询，在查询值两侧添加 `%` 通配符。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [utils.go](utils.go) | 通用工具函数 |
| [crypto.go](crypto.go) | 加密工具 |
| [websocket.go](websocket.go) | WebSocket 工具 |
| [smtp/format.go](smtp/format.go) | 格式化 |
| [smtp/smtpool.go](smtp/smtpool.go) | SMTP 连接池 |
| [m3u8/m3u8.go](m3u8/m3u8.go) | M3U8 解析 |
| [fastJSONSerializer/fastJSONSerializer.go](fastJSONSerializer/fastJSONSerializer.go) | JSON 序列化器 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
