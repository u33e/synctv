# [根目录](../CLAUDE.md) > internal/email (邮件服务)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / email (邮件服务)

## 模块职责

`internal/email` 模块提供邮件发送功能，用于发送验证码、注册通知和找回密码邮件。使用 MJML 模板生成美观的 HTML 邮件。

## 目录结构

```
internal/email/
├── email.go            # 邮件发送核心逻辑
├── smtp.go            # SMTP 连接池
└── emailtemplate/     # MJML 邮件模板
    ├── embed.go       # 模板嵌入
    ├── captcha.mjml   # 验证码模板
    ├── test.mjml      # 测试邮件模板
    └── retrieve_password.mjml # 找回密码模板
```

## 入口与启动

### 初始化

邮件服务通过设置系统控制启用状态:

```go
var EnableEmail = settings.NewBoolSetting(
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

当 `EnableEmail` 为 `false` 时，所有邮件操作返回 `ErrEmailNotEnabled`。

## 核心功能

### 验证码缓存

使用内存缓存存储验证码，有效期 5 分钟:

```go
var emailCaptcha *synccache.SyncCache[string, string]
```

### 验证码类型

| 类型 | 缓存键格式 |
|------|-----------|
| 注册验证码 | `signup:{email}` |
| 绑定邮箱验证码 | `bind:{userID}:{email}` |
| 找回密码验证码 | `retrieve_password:{userID}:{email}` |

## 对外接口

### 发送测试邮件

```go
func SendTestEmail(username, email string) error
```

**用途**: 测试邮件配置是否正确

### 注册相关

**发送注册验证码**:

```go
func SendSignupCaptchaEmail(email string) error
```

**验证注册验证码**:

```go
func VerifySignupCaptchaEmail(email, captcha string) (bool, error)
```

### 邮箱绑定相关

**发送绑定验证码**:

```go
func SendBindCaptchaEmail(userID, userEmail string) error
```

**验证绑定验证码**:

```go
func VerifyBindCaptchaEmail(userID, userEmail, captcha string) (bool, error)
```

### 找回密码相关

**发送找回密码验证码**:

```go
func SendRetrievePasswordCaptchaEmail(userID, email, host string) error
```

生成的邮件包含重置密码链接。

**验证找回密码验证码**:

```go
func VerifyRetrievePasswordCaptchaEmail(userID, email, captcha string) (bool, error)
```

## 邮件模板

### Captcha 模板 (验证码)

包含 6 位验证码显示。

### Test 模板 (测试)

包含用户名和年份信息，用于测试邮件发送。

### Retrieve Password 模板 (找回密码)

包含:
- 验证码
- 重置密码链接
- 主机地址

## 邮件设置

### 通过设置系统配置

| 设置名称 | 类型 | 默认值 | 说明 |
|----------|------|--------|------|
| `enable_email` | bool | `false` | 是否启用邮件 |
| `email_disable_user_signup` | bool | `false` | 禁用用户注册 |
| `email_signup_need_review` | bool | `false` | 注册需要审核 |
| `email_signup_white_list_enable` | bool | `false` | 启用邮箱白名单 |
| `email_signup_white_list` | string | 默认列表 | 允许注册的邮箱域名 |

### 白名单验证

当 `email_signup_white_list_enable` 为 `true` 时，只允许白名单邮箱注册。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/Boostport/mjml-go` | MJML 转 HTML |
| `github.com/synctv-org/synctv/internal/email/emailtemplate` | 邮件模板 |
| `github.com/synctv-org/synctv/internal/settings` | 设置系统 |
| `github.com/synctv-org/synctv/internal/model` | 模型定义 |
| `github.com/zijiren233/gencontainer/synccache` | 内存缓存 |

## 使用示例

### 发送注册验证码

```go
err := email.SendSignupCaptchaEmail("user@example.com")
if err != nil {
    log.Errorf("send email failed: %v", err)
}
```

### 验证验证码

```go
valid, err := email.VerifySignupCaptchaEmail("user@example.com", "123456")
if err != nil || !valid {
    // 验证码无效或过期
}
```

## 常见问题 (FAQ)

### 如何启用邮件功能?

1. 设置 SMTP 配置 (通过管理后台)
2. 启用邮件设置:

```bash
synctv setting set enable_email true
```

### 验证码有效期是多久?

验证码有效期为 5 分钟，验证后自动删除。

### 邮件模板如何修改?

修改 `internal/email/emailtemplate/*.mjml` 文件，重新编译即可生效。

### 测试邮件功能

通过管理后台或 CLI 发送测试邮件:

```bash
synctv setting test-email user@example.com
```

## 错误处理

| 错误 | 说明 |
|------|------|
| `ErrEmailNotEnabled` | 邮件功能未启用 |

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [email.go](email.go) | 邮件发送核心逻辑 |
| [smtp.go](smtp.go) | SMTP 连接池 |
| [emailtemplate/embed.go](emailtemplate/embed.go) | 模板嵌入 |
| [emailtemplate/captcha.mjml](emailtemplate/captcha.mjml) | 验证码模板 |
| [emailtemplate/test.mjml](emailtemplate/test.mjml) | 测试邮件模板 |
| [emailtemplate/retrieve_password.mjml](emailtemplate/retrieve_password.mjml) | 找回密码模板 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
