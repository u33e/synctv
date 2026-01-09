# [根目录](../CLAUDE.md) > internal/captcha (验证码服务)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / captcha (验证码服务)

## 模块职责

`internal/captcha` 模块提供图形验证码功能，用于防止自动化攻击。基于 [base64Captcha](https://github.com/mojocn/base64Captcha) 实现。

## 目录结构

```
internal/captcha/
└── captcha.go    # 验证码服务
```

## 入口与启动

验证码服务在包初始化时自动启动:

```go
var Captcha *base64Captcha.Captcha

func init() {
    Captcha = base64Captcha.NewCaptcha(
        base64Captcha.DefaultDriverDigit,
        base64Captcha.DefaultMemStore,
    )
}
```

## 核心功能

### 验证码生成器

使用 `DefaultDriverDigit` 数字驱动器:
- 生成 6 位数字验证码
- 高度 80 像素
- 宽度 240 像素

### 验证码存储

使用 `DefaultMemStore` 内存存储，验证码有效期默认为配置的时间。

## 对外接口

该模块不提供直接的对外接口，通过 HTTP 处理器 (`server/handlers`) 暴露 API。

### 验证码 API

- `POST /api/captcha/generate` - 生成验证码
- `POST /api/captcha/verify` - 验证验证码

## 使用示例

### 生成验证码

```go
id, b64s, err := captcha.Captcha.Generate()
if err != nil {
    return err
}

// 返回给客户端
// id: 验证码 ID
// b64s: base64 编码的图片
```

### 验证验证码

```go
verified := captcha.Captcha.Verify(id, answer, true)
```

参数:
- `id`: 验证码 ID
- `answer`: 用户输入的答案
- `true`: 验证后删除验证码

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `github.com/mojocn/base64Captcha` | 验证码生成 |

## 配置

当前使用默认配置:
- 驱动: 数字驱动器
- 存储: 内存存储
- 高度: 80
- 宽度: 240
- 验证码长度: 6

## 常见问题 (FAQ)

### 如何自定义验证码样式?

需要修改驱动器配置。当前使用默认数字驱动器。

### 验证码有效期多久?

由存储实现决定，默认内存存储中没有显式设置有效期。

### 验证码存储在哪里?

使用内存存储，重启后验证码会失效。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [captcha.go](captcha.go) | 验证码服务 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
