# [根目录](../CLAUDE.md) > internal/settings (动态设置系统)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / settings (动态设置系统)

## 模块职责

`internal/settings` 模块提供运行时动态配置系统，允许在不需要重启服务的情况下修改配置。设置持久化到数据库，支持按分组管理。

## 目录结构

```
internal/settings/
├── setting.go    # 设置核心接口和基础实现
├── bool.go       # 布尔设置
├── string.go     # 字符串设置
├── int64.go      # 整数设置
└── floate64.go   # 浮点数设置
```

## 设置类型

支持的设置类型:

| 类型 | 说明 | 接口 |
|------|------|------|
| BoolSetting | 布尔值 | `Set(bool) error`, `Get() bool` |
| StringSetting | 字符串 | `Set(string) error`, `Get() string` |
| Int64Setting | 64 位整数 | `Set(int64) error`, `Get() int64` |
| Float64Setting | 浮点数 | `Set(float64) error`, `Get() float64` |

## 设置分组

通过 `model.SettingGroup` 分组管理设置:

| 分组 | 说明 |
|------|------|
| `SettingGroupServer` | 服务器相关设置 |
| `SettingGroupEmail` | 邮件相关设置 |
| `SettingGroupOAuth2` | OAuth2 相关设置 |

## 核心接口

### Setting 接口

```go
type Setting interface {
    Name() string
    Type() model.SettingType
    Group() model.SettingGroup
    Init(value string) error
    Inited() bool
    SetInitPriority(priority int)
    InitPriority() int
    String() string
    SetString(value string) error
    DefaultString() string
    DefaultInterface() any
    Interface() any
}
```

### 创建设置

**布尔设置**:

```go
func NewBoolSetting(
    name string,
    defaultValue bool,
    group model.SettingGroup,
    opts ...BoolSettingOption,
) BoolSetting
```

**选项**:
- `WithBeforeInitBool` - 初始化前钩子
- `WithAfterInitBool` - 初始化后钩子
- `WithBeforeSetBool` - 设置前钩子
- `WithAfterSetBool` - 设置后钩子

**示例**:

```go
EnableEmail = settings.NewBoolSetting(
    "enable_email",
    false,
    model.SettingGroupEmail,
    settings.WithAfterSetBool(func(_ settings.BoolSetting, b bool) {
        if !b {
            closeSMTPPool()
        }
    }),
)
```

**字符串设置**:

```go
func NewStringSetting(
    name string,
    defaultValue string,
    group model.SettingGroup,
    opts ...StringSettingOption,
) StringSetting
```

**整数设置**:

```go
func NewInt64Setting(
    name string,
    defaultValue int64,
    group model.SettingGroup,
    opts ...Int64SettingOption,
) Int64Setting
```

**浮点数设置**:

```go
func NewFloat64Setting(
    name string,
    defaultValue float64,
    group model.SettingGroup,
    opts ...Float64SettingOption,
) Float64Setting
```

## 通用操作

### 动态设置值

```go
func SetValue(name string, value any) error
```

支持多种输入类型:
- `bool` -> BoolSetting
- `int`/`int64` -> Int64Setting
- `float`/`float64` -> Float64Setting
- `string` -> StringSetting

### 初始化设置

设置按优先级初始化 (通过堆排序):

```go
func PopNeedInit() (Setting, bool)
```

## 系统设置示例

### 服务器设置

```go
Version = settings.NewStringSetting(
    "version",
    "dev",
    model.SettingGroupServer,
)
```

### 邮件设置

```go
EnableEmail = settings.NewBoolSetting(
    "enable_email",
    false,
    model.SettingGroupEmail,
)
DisableUserSignup = settings.NewBoolSetting(
    "email_disable_user_signup",
    false,
    model.SettingGroupEmail,
)
SignupNeedReview = settings.NewBoolSetting(
    "email_signup_need_review",
    false,
    model.SettingGroupEmail,
)
EmailSignupWhiteListEnable = settings.NewBoolSetting(
    "email_signup_white_list_enable",
    false,
    model.SettingGroupEmail,
)
EmailSignupWhiteList = settings.NewStringSetting(
    "email_signup_white_list",
    `gmail.com,qq.com,163.com,yahoo.com,sina.com,126.com,outlook.com,yeah.net,foxmail.com`,
    model.SettingGroupEmail,
)
```

## 对外接口

该模块不提供直接的对外 API，通过管理接口 (`server/handlers/admin.go`) 进行修改。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/synctv-org/synctv/internal/model` | 设置模型 |
| `github.com/zijiren233/gencontainer/heap` | 堆 (优先级初始化) |

## 使用示例

### 创建自定义设置

```go
var MySetting = settings.NewBoolSetting(
    "my_setting_name",
    false,
    model.SettingGroupServer,
    settings.WithAfterSetBool(func(s settings.BoolSetting, b bool) {
        log.Infof("setting %s set to %v", s.Name(), b)
    }),
)
```

### 读取设置值

```go
if MySetting.Get() {
    // 启用状态
}
```

### 设置值

```go
err := MySetting.Set(true)
```

## 常见问题 (FAQ)

### 如何确保设置初始化顺序?

使用 `SetInitPriority` 设置优先级，数值越大优先级越高:

```go
settings.SetInitPriority(100)
```

### 如何阻止某些设置被修改?

使用 `WithBeforeSetBool` 钩子:

```go
Version = settings.NewStringSetting(
    "version",
    "dev",
    model.SettingGroupServer,
    settings.WithBeforeSetString(func(_ settings.StringSetting, _ string) (string, error) {
        return "", errors.New("version can not be set")
    }),
)
```

### 设置持久化到哪里?

所有设置都会保存到 `settings` 数据库表中。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [setting.go](setting.go) | 设置核心接口 |
| [bool.go](bool.go) | 布尔设置 |
| [string.go](string.go) | 字符串设置 |
| [int64.go](int64.go) | 整数设置 |
| [floate64.go](floate64.go) | 浮点数设置 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
