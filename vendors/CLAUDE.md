# [根目录](../CLAUDE.md) > vendors (厂商后端服务)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / vendors (厂商后端服务)

## 模块概述

`vendors` 是 SyncTV 的厂商后端微服务，基于 Kratos 框架实现。提供 Bilibili、Alist、Emby 等第三方服务的统一 API 接口，通过 gRPC 与主服务通信。

## 技术栈

| 技术 | 用途 |
|------|------|
| Go | 编程语言 |
| Kratos | 微服务框架 |
| gRPC | 通信协议 |
| OpenAPI | HTTP API 文档 |
| Wire | 依赖注入 |

## 目录结构

```
vendors/
├── cmd/                     # 命令行
│   ├── root.go              # 根命令
│   └── server/             # Server 命令
│       ├── main.go         # 入口
│       ├── wire.go         # Wire 生成配置
│       └── wire_gen.go    # Wire 生成代码
├── internal/                # 内部包
│   ├── server/            # gRPC/HTTP 服务器
│   │   ├── server.go      # 服务器实现
│   │   ├── all.go        # 服务注册
│   │   ├── alist/        # Alist 服务
│   │   ├── bilibili/     # Bilibili 服务
│   │   └── emby/        # Emby 服务
│   ├── bootstrap/         # 启动初始化
│   │   └── log.go        # 日志初始化
│   └── registry/          # 服务发现
│       ├── registry.go    # 注册中心接口
│       ├── consul.go      # Consul 实现
│       └── etcd.go       # etcd 实现
├── api/                    # Protobuf 定义
│   ├── alist/            # Alist API
│   │   ├── alist.proto
│   │   ├── alist.pb.go
│   │   ├── alist_grpc.pb.go
│   │   └── alist_http.pb.go
│   ├── bilibili/         # Bilibili API
│   │   ├── bilibili.proto
│   │   ├── bilibili.pb.go
│   │   ├── bilibili_grpc.pb.go
│   │   └── bilibili_http.pb.go
│   ├── emby/             # Emby API
│   │   ├── emby.proto
│   │   ├── emby.pb.go
│   │   ├── emby_grpc.pb.go
│   │   └── emby_http.pb.go
│   └── webdav/           # WebDAV API
│       ├── webdav.proto
│       ├── webdav.pb.go
│       ├── webdav_grpc.pb.go
│       └── webdav_http.pb.go
├── service/                # 服务实现
│   ├── all.go            # 服务集合
│   ├── alist/            # Alist 服务实现
│   │   ├── alist.go
│   │   └── service.go
│   ├── bilibili/         # Bilibili 服务实现
│   │   ├── bilibili.go
│   │   └── service.go
│   ├── emby/             # Emby 服务实现
│   │   ├── emby.go
│   │   └── service.go
│   └── webdav/           # WebDAV 服务实现
│       ├── service.go
│       └── webdav.go
├── vendors/                # 厂商客户端
│   ├── alist/            # Alist 客户端
│   │   ├── alist.go
│   │   ├── client.go
│   │   ├── fs.go
│   │   ├── login.go
│   │   └── me.go
│   ├── bilibili/         # Bilibili 客户端
│   │   ├── bilibili.go
│   │   ├── client.go
│   │   ├── live.go
│   │   ├── login.go
│   │   ├── movie.go
│   │   ├── user.go
│   │   ├── utils.go
│   │   ├── wbi.go
│   │   └── buvid.go
│   ├── emby/             # Emby 客户端
│   │   ├── emby.go
│   │   ├── client.go
│   │   ├── library.go
│   │   ├── login.go
│   │   ├── me.go
│   │   ├── system.go
│   │   └── user.go
│   └── onedrive/         # OneDrive 客户端
│       ├── onedrive.go
│       ├── client.go
│       ├── drive.go
│       ├── fs.go
│       ├── login.go
│       └── user.go
├── conf/                   # 配置
│   ├── conf.proto       # 配置定义
│   ├── conf.pb.go       # 生成的配置
│   └── default.go      # 默认配置
├── utils/                  # 工具
│   └── utils.go
├── third_party/            # 第三方依赖
│   ├── google/          # Google Protobuf
│   ├── openapi/         # OpenAPI 注解
│   └── validate/        # 验证规则
├── openapi.yaml           # OpenAPI 文档
├── Makefile               # 构建脚本
├── Dockerfile             # Docker 镜像
└── main.go                # 程序入口
```

## 服务列表

### Alist Service

**功能**:
- 文件列表浏览
- 文件下载代理
- 目录操作

### Bilibili Service

**功能**:
- 直播流获取
- 视频播放
- 弹幕获取
- 扫码登录

### Emby Service

**功能**:
- 媒体库浏览
- 视频播放
- 用户认证

### WebDAV Service

**功能**:
- WebDAV 协议支持
- 文件操作

## API 接口

### gRPC 接口

通过 gRPC 提供高性能服务调用。

### HTTP 接口

通过 OpenAPI 提供 HTTP/JSON 接口，方便调试和集成。

## 开发命令

### 初始化

```bash
cd vendors
make init
```

### 生成代码

```bash
make config    # 生成配置 Protobuf
make api       # 生成 API Protobuf
make generate  # 生成 Wire 依赖注入
make all       # 生成所有
```

### 构建

```bash
make build     # 构建当前平台
make run       # 构建并运行
```

### 运行

```bash
# 开发模式
go run cmd/server/main.go

# 生产模式
./bin/vendors -conf ./configs
```

## 配置

### 服务发现

支持以下服务发现方式:

1. **直接连接**: 指定端点地址
2. **Consul**: 使用 Consul 服务注册发现
3. **etcd**: 使用 etcd 服务注册发现

### 环境变量

支持通过 `.env` 文件配置:

```bash
# 服务端口
SERVER_HTTP_PORT=8000
SERVER_GRPC_PORT=9000

# 服务发现
CONSUL_ADDRESS=localhost:8500
ETCD_ENDPOINTS=localhost:2379
```

## 部署

### Docker

```bash
docker build -t synctv-vendors:latest .
docker run -p 8000:8000 -p 9000:9000 synctv-vendors:latest
```

### Kubernetes

使用 Helm Chart 部署。

## 对外接口

### Alist 接口

```go
service Alist {
    rpc GetFs(GetFsReq) returns (GetFsResp);
    rpc Login(LoginReq) returns (LoginResp);
    rpc Me(MeReq) returns (MeResp);
}
```

### Bilibili 接口

```go
service Bilibili {
    rpc Parse(ParseReq) returns (ParseResp);
    rpc Login(LoginReq) returns (LoginResp);
    rpc Me(MeReq) returns (MeResp);
    rpc GetLiveStream(GetLiveStreamReq) returns (GetLiveStreamResp);
    rpc GetDanmu(GetDanmuReq) returns (GetDanmuResp);
}
```

### Emby 接口

```go
service Emby {
    rpc List(ListReq) returns (ListResp);
    rpc Login(LoginReq) returns (LoginResp);
    rpc Me(MeReq) returns (MeResp);
}
```

### WebDAV 接口

```go
service WebDAV {
    rpc GetFs(GetFsReq) returns (GetFsResp);
}
```

## Wire 依赖注入

使用 Wire 自动生成依赖注入代码:

```bash
make generate
```

生成的代码在 `cmd/server/wire_gen.go`。

## 常见问题 (FAQ)

### 如何添加新的厂商服务?

1. 在 `vendors/` 下创建新的客户端
2. 在 `service/` 下创建服务实现
3. 在 `api/` 下创建 Protobuf 定义
4. 在 `internal/server/` 下注册服务
5. 运行 `make all` 生成代码

### 如何调试 API?

使用 OpenAPI 文档 (`openapi.yaml`) 或使用 gRPC 客户端工具 (如 grpcurl)。

### 如何配置服务发现?

在配置文件中设置 Consul 或 etcd 地址，服务启动时会自动注册。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [main.go](main.go) | 程序入口 |
| [cmd/root.go](cmd/root.go) | 根命令 |
| [cmd/server/main.go](cmd/server/main.go) | Server 入口 |
| [Makefile](Makefile) | 构建脚本 |
| [openapi.yaml](openapi.yaml) | OpenAPI 文档 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
