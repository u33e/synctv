# [根目录](../CLAUDE.md) > public (静态资源)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / public (静态资源)

## 模块职责

`public` 模块负责前端静态资源的嵌入和管理。通过 Go 的 `embed` 指令将前端构建产物嵌入到二进制文件中。

## 目录结构

```
public/
└── public.go           # 静态资源嵌入
```

## 核心功能

### 静态资源嵌入

```go
//go:embed all:dist
var dist embed.FS

var Public, _ = fs.Sub(dist, "dist")
```

- 使用 `embed.FS` 嵌入整个 `dist/` 目录
- 通过 `fs.Sub()` 创建子文件系统

## 资源目录

- `dist/` - 前端构建产物 (由 `synctv-web` 生成)

## 使用方式

### 嵌入到二进制

```go
import (
    "github.com/synctv-org/synctv/public"
    "io/fs"
)

// 使用嵌入的静态资源
httpFS := http.FS(public.Public)
http.FileServer(httpFS)
```

### 配置 Web 路径

通过 `--web-path` 参数指定自定义 Web 路径，否则使用嵌入的静态资源。

## 构建流程

```
1. 构建前端
   cd synctv-web
   pnpm build

2. 复制产物到 public/dist
   (syntv-web/dist -> public/dist)

3. 编译 Go 程序 (自动嵌入)
   go build
```

## 对外接口

该模块不提供对外 API，通过 `server/static/` 模块提供 HTTP 访问。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `embed` | Go 标准库，资源嵌入 |
| `io/fs` | 文件系统接口 |

## 常见问题 (FAQ)

### 如何更新前端?

1. 重新构建前端: `cd synctv-web && pnpm build`
2. 重新编译: `go build`

### 如何禁用 Web 前端?

```bash
synctv server --disable-web
```

### 如何使用外部静态文件?

```bash
synctv server --web-path /path/to/dist
```

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [public.go](public.go) | 静态资源嵌入 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
