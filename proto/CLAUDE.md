# [根目录](../CLAUDE.md) > proto (协议定义)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / proto (协议定义)

## 模块职责

`proto` 模块定义 SyncTV 的通信协议，使用 Protobuf 格式。包含 WebSocket 消息定义和 OAuth2 插件接口定义。

## 目录结构

```
proto/
├── message/            # WebSocket 消息定义
│   ├── message.proto   # 消息定义
│   ├── message.pb.go   # 生成的 Go 代码
│   └── message.go     # 消息封装
└── provider/           # OAuth2 提供商定义
    ├── plugin.proto    # 插件接口定义
    ├── plugin.pb.go   # 生成的 Go 代码
    └── plugin_grpc.pb.go # 生成的 gRPC 代码
```

## WebSocket 消息 (message/)

### MessageType (消息类型)

| 类型 | 值 | 说明 |
|------|------|------|
| UNKNOWN | 0 | 未知消息 |
| ERROR | 1 | 错误消息 |
| CHAT | 2 | 聊天消息 |
| STATUS | 3 | 播放状态 |
| CHECK_STATUS | 4 | 检查状态 |
| EXPIRED | 5 | 会话过期 |
| CURRENT | 6 | 当前播放信息 |
| MOVIES | 7 | 影片列表 |
| VIEWER_COUNT | 8 | 观看人数 |
| SYNC | 9 | 同步指令 |
| MY_STATUS | 10 | 我的播放状态 |
| WEBRTC_OFFER | 11 | WebRTC Offer |
| WEBRTC_ANSWER | 12 | WebRTC Answer |
| WEBRTC_ICE_CANDIDATE | 13 | WebRTC ICE 候选 |
| WEBRTC_JOIN | 14 | WebRTC 加入 |
| WEBRTC_LEAVE | 15 | WebRTC 离开 |

### Message (消息结构)

```protobuf
message Message {
    MessageType type = 1;              // 消息类型
    sfixed64 timestamp = 2;            // 时间戳
    optional Sender sender = 3;        // 发送者

    oneof payload {
        string error_message = 4;      // 错误信息 (ERROR)
        string chat_content = 5;       // 聊天内容 (CHAT)
        Status playback_status = 6;    // 播放状态 (STATUS/CURRENT/SYNC)
        fixed64 expiration_id = 7;     // 会话 ID (EXPIRED)
        int64 viewer_count = 8;        // 观看人数 (VIEWER_COUNT)
        WebRTCData webrtc_data = 9;     // WebRTC 数据 (WEBRTC_*)
    }
}
```

### Sender (发送者信息)

```protobuf
message Sender {
    string user_id = 1;    // 用户 ID
    string username = 2;    // 用户名
}
```

### Status (播放状态)

```protobuf
message Status {
    bool is_playing = 1;        // 是否播放中
    double current_time = 2;    // 当前时间 (秒)
    double playback_rate = 3;    // 播放速度
}
```

### WebRTCData (WebRTC 数据)

```protobuf
message WebRTCData {
    string data = 1;   // SDP 数据
    string to = 2;     // 目标用户 ID
    string from = 3;    // 源用户 ID
}
```

## OAuth2 插件接口 (provider/)

### Oauth2Plugin 服务

```protobuf
service Oauth2Plugin {
    // 初始化插件
    rpc Init(InitReq) returns (Enpty) {}

    // 获取提供商名称
    rpc Provider(Enpty) returns (ProviderResp) {}

    // 生成认证 URL
    rpc NewAuthURL(NewAuthURLReq) returns (NewAuthURLResp) {}

    // 获取用户信息
    rpc GetUserInfo(GetUserInfoReq) returns (GetUserInfoResp) {}
}
```

### 请求/响应消息

```protobuf
// 初始化请求
message InitReq {
    string client_id = 1;
    string client_secret = 2;
    string redirect_url = 3;
}

// 获取 Token 请求
message GetTokenReq {
    string code = 1;
}

// 刷新 Token 请求
message RefreshTokenReq {
    string refresh_token = 1;
}

// 提供商响应
message ProviderResp {
    string name = 1;
}

// 生成认证 URL 请求
message NewAuthURLReq {
    string state = 1;
}

// 生成认证 URL 响应
message NewAuthURLResp {
    string url = 1;
}

// 获取用户信息请求
message GetUserInfoReq {
    string code = 1;
}

// 获取用户信息响应
message GetUserInfoResp {
    string username = 1;
    string provider_user_id = 2;
}

// 空消息
message Enpty {}
```

## 代码生成

### 生成 Go 代码

```bash
cd proto
protoc --go_out=. --go_opt=paths=source_relative \
    message/message.proto \
    provider/plugin.proto
```

### 生成 gRPC 代码

```bash
protoc --go_out=. --go-grpc_out=. \
    --go_opt=paths=source_relative \
    --go-grpc_opt=paths=source_relative \
    provider/plugin.proto
```

或使用项目脚本:

```bash
./script/proto.sh
```

## 使用示例

### 发送聊天消息

```go
msg := pb.Message{
    Type:      pb.MessageType_CHAT,
    Timestamp: time.Now().UnixNano(),
    Sender: &pb.Sender{
        UserId:   userID,
        Username: username,
    },
    Payload: &pb.Message_ChatContent{
        ChatContent: "Hello, world!",
    },
}
```

### 发送播放状态

```go
msg := pb.Message{
    Type:      pb.MessageType_STATUS,
    Timestamp: time.Now().UnixNano(),
    Sender: &pb.Sender{
        UserId:   userID,
        Username: username,
    },
    Payload: &pb.Message_PlaybackStatus{
        PlaybackStatus: &pb.Status{
            IsPlaying:    true,
            CurrentTime:  123.45,
            PlaybackRate: 1.0,
        },
    },
}
```

### 发送 WebRTC Offer

```go
msg := pb.Message{
    Type:      pb.MessageType_WEBRTC_OFFER,
    Timestamp: time.Now().UnixNano(),
    Payload: &pb.Message_WebrtcData{
        WebrtcData: &pb.WebRTCData{
            Data: sdpOffer,
            To:   targetUserID,
            From:  myUserID,
        },
    },
}
```

## 消息类型说明

### 聊天消息 (CHAT)

用于房间内聊天，包含发送者信息和聊天内容。

### 播放状态 (STATUS/CURRENT/SYNC)

- `STATUS`: 播放状态变更
- `CURRENT`: 当前播放信息
- `SYNC`: 同步指令

### WebRTC 消息 (WEBRTC_*)

用于语音通话:
- `WEBRTC_OFFER`: SDP Offer
- `WEBRTC_ANSWER`: SDP Answer
- `WEBRTC_ICE_CANDIDATE`: ICE 候选
- `WEBRTC_JOIN`: 加入通话
- `WEBRTC_LEAVE`: 离开通话

### 观看人数 (VIEWER_COUNT)

实时显示房间观看人数。

### 会话过期 (EXPIRED)

当会话过期时发送，要求客户端重新登录。

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `google.golang.org/protobuf` | Protobuf 运行时 |

## 常见问题 (FAQ)

### 消息时间戳精度?

使用 `sfixed64` 类型，精度为纳秒。

### 如何添加新的消息类型?

1. 在 `message.proto` 中添加 `MessageType` 枚举值
2. 在 `Message` 的 `oneof payload` 中添加新字段
3. 重新生成代码

### WebRTC 支持情况?

目前支持语音通话，视频和屏幕共享开发中。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [message/message.proto](message/message.proto) | 消息定义 |
| [message/message.pb.go](message/message.pb.go) | 生成的 Go 代码 |
| [message/message.go](message/message.go) | 消息封装 |
| [provider/plugin.proto](provider/plugin.proto) | 插件接口定义 |
| [provider/plugin.pb.go](provider/plugin.pb.go) | 生成的 Go 代码 |
| [provider/plugin_grpc.pb.go](provider/plugin_grpc.pb.go) | 生成的 gRPC 代码 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
