# SyncTV 屏幕共享功能实施规划 (v2.0)

> 规划版本: v2.0 (根据用户反馈更新)
> 创建日期: 2026-01-10
> 更新日期: 2026-01-10
> 状态: 待确认

---

## 用户需求澄清

| 问题 | 用户选择 | 实现方案 |
|------|----------|----------|
| 视频显示位置 | 集成到添加影片源中 | 新增"屏幕共享"影片类型，在主播放器播放 |
| 音频处理 | 默认共享音频，用户可选择关闭 | 默认包含系统音频，提供关闭开关 |
| 多屏幕共享 | 无此问题（集成到影片源） | 同一时间一个共享源，如现有 RTMP 直播 |
| 视频质量控制 | 用户手动选择多种质量 | 提供高清/标清/流畅选项 |

---

## 技术方案概述

### 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          前端 (浏览器)                              │
│  ┌─────────────────┐      ┌─────────────────┐      ┌──────────┐  │
│  │  屏幕捕获       │────▶│  MediaRecorder │────▶│  FLV     │  │
│  │  getDisplay    │      │  编码为 FLV    │      │  推流器  │  │
│  │  Media()       │      │                 │      │          │  │
│  └─────────────────┘      └─────────────────┘      └────┬─────┘  │
│                                                              │        │
└──────────────────────────────────────────────────────────────────────│────────┘
                                                               │ RTMP
                                                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          后端 (SyncTV)                            │
│  ┌─────────────────┐      ┌─────────────────┐      ┌──────────┐  │
│  │  RTMP Server   │────▶│   livelib      │────▶│  HLS/   │  │
│  │  (已有)        │      │   Channel       │      │  FLV     │  │
│  │               │      │   (转码/分发)   │      │  播放接口│  │
│  └─────────────────┘      └─────────────────┘      └──────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────────────┘
                                                                   │
                                                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      观众端 (浏览器)                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  主播放器 (ArtPlayer) - 播放 HLS/FLV 流                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 技术选型

| 组件 | 技术 | 说明 |
|--------|------|------|
| 屏幕捕获 | `navigator.mediaDevices.getDisplayMedia()` | 浏览器原生 API |
| 流编码 | `MediaRecorder` → FLV | 实时编码为 FLV 格式 |
| 推流协议 | RTMP (通过 WebSocket/HTTP) | 复用现有 RTMP 服务器 |
| 后端流处理 | livelib Channel (已有) | 转换为 HLS/FLV |
| 播放格式 | HLS (.m3u8) / FLV | 复用现有播放器 |

---

## 后端改动

### 1. 数据模型扩展

**文件**: [internal/model/movie.go](../internal/model/movie.go)

```go
// 在 MovieBase 中添加字段
type MovieBase struct {
    // ... 现有字段 ...

    // 屏幕共享相关配置
    ScreenShareConfig *ScreenShareConfig `gorm:"serializer:fastjson;type:text" json:"screenShareConfig,omitempty"`
}

type ScreenShareConfig struct {
    IncludeAudio    bool   `json:"includeAudio"`     // 是否包含音频
    Quality        string  `json:"quality"`          // 质量选项: high/medium/low
    FrameRate      uint32  `json:"frameRate"`        // 帧率: 30/15
    Bitrate        uint32  `json:"bitrate"`          // 码率 (kbps)
}
```

### 2. 影片源类型扩展

**文件**: [internal/model/movie.go](../internal/model/movie.go)

```go
const (
    VendorBilibili VendorName = "bilibili"
    VendorAlist    VendorName = "alist"
    VendorEmby     VendorName = "emby"
    VendorScreenShare VendorName = "screencast" // 新增
)
```

**文件**: [internal/op/movie.go](../internal/op/movie.go)

```go
func (m *Movie) initChannel() (*rtmps.Channel, error) {
    if !m.Live || (!m.RtmpSource && !m.Proxy) {
        return nil, errors.New("this movie not support channel")
    }

    if m.RtmpSource {
        return m.initRtmpSourceChannel()
    }

    // 处理屏幕共享流
    if m.VendorInfo.Vendor == model.VendorScreenShare {
        return m.initRtmpSourceChannel() // 复用 RTMP source 逻辑
    }

    // Handle proxy case
    // ... 现有代码
}
```

### 3. API 处理器更新

**文件**: [server/handlers/movie.go](../server/handlers/movie.go)

```go
func genMovieInfo(...) (*model.Movie, error) {
    // ... 现有代码

    switch {
    case movie.VendorInfo.Vendor == model.VendorScreenShare:
        // 屏幕共享流，生成播放 URL
        movie.URL = fmt.Sprintf(
            "/api/room/movie/live/hls/list/%s.m3u8?token=%s&roomId=%s",
            movie.ID,
            userToken,
            opMovie.RoomID,
        )
        movie.Type = "m3u8"
        movie.Live = true
        movie.MoreSources = append(movie.MoreSources, &dbModel.MoreSource{
            Name: "flv",
            URL: fmt.Sprintf(
                "/api/room/movie/live/flv/%s.flv?token=%s&roomId=%s",
                movie.ID,
                userToken,
                opMovie.RoomID,
            ),
            Type: "flv",
        })
    // ... 其他 case
    }
}
```

### 4. 路由配置

**文件**: [server/router.go](../server/router.go)

无需新增路由，复用现有的：
- `POST /api/room/movie/push` - 添加屏幕共享影片
- `POST /api/room/movie/newPublishKey` - 获取推流密钥（可选）
- `GET /api/room/movie/live/hls/list/:movieId.m3u8` - HLS 播放
- `GET /api/room/movie/live/flv/:movieId.flv` - FLV 播放

---

## 前端改动

### 1. MoviePush 组件扩展

**文件**: [synctv-web/src/components/cinema/MoviePush.vue](../synctv-web/src/components/cinema/MoviePush.vue)

```typescript
enum pushType {
  MOVIE = 0,
  DIR,
  LIVE,
  RTMP_SOURCE,
  BILIBILI,
  ALIST,
  EMBY,
  SCREEN_SHARE = 8,  // 新增
}

const movieTypeRecords: Map<pushType, movieTypeRecord> = new Map([
  // ... 现有类型
  [
    pushType.SCREEN_SHARE,
    {
      name: "屏幕共享",
      comment: "共享屏幕到房间",
      advanced: true,
      showProxy: false,
      defaultType: "m3u8",
      allowedTypes: []
    }
  ]
]);

// 屏幕共享配置
const screenShareConfig = ref({
  includeAudio: true,  // 默认包含音频
  quality: "high",     // high/medium/low
  frameRate: 30,
  bitrate: 2000
});

const qualityOptions = [
  { name: "高清", value: "high", bitrate: 2000, frameRate: 30 },
  { name: "标清", value: "medium", bitrate: 1000, frameRate: 15 },
  { name: "流畅", value: "low", bitrate: 500, frameRate: 15 }
];

case pushType.SCREEN_SHARE:
  newMovieInfo.value = {
    url: "",
    name: newMovieInfo.value.name,
    type: "m3u8",
    proxy: false,
    live: true,
    rtmpSource: false,  // 不是 RTMP 推流源
    headers: {},
    vendorInfo: {
      vendor: "screencast",
      screenShareConfig: { ...screenShareConfig.value }
    }
  };
  break;
```

**UI 更新**:

```vue
<!-- 高级选项中添加质量选择 -->
<div class="more-option-list" v-if="selectedMovieType === pushType.SCREEN_SHARE">
  <span class="text-sm min-w-fit"> 视频质量： </span>
  <select v-model="screenShareConfig.quality" class="bg-transparent p-0 text-base w-full h-5">
    <option v-for="opt in qualityOptions" :value="opt.value">
      {{ opt.name }}
    </option>
  </select>
</div>

<div class="more-option-list" v-if="selectedMovieType === pushType.SCREEN_SHARE">
  <label>共享系统音频：
    <input type="checkbox" v-model="screenShareConfig.includeAudio" />
  </label>
</div>

<!-- 添加按钮显示 -->
<button
  v-if="selectedMovieType === pushType.SCREEN_SHARE"
  class="btn"
  @click="startScreenShare"
>
  开始共享
</button>
```

### 2. 新增 FLV 推流器组件

**文件**: [synctv-web/src/components/cinema/ScreenSharePublisher.vue](../synctv-web/src/components/cinema/ScreenSharePublisher.vue) (新建)

功能：
- 获取屏幕流 `getDisplayMedia()`
- MediaRecorder 编码为 FLV
- 通过 WebSocket/HTTP 推送到服务器 RTMP 端口

```typescript
// 关键代码结构
class FLVStreamPublisher {
  private mediaRecorder?: MediaRecorder;
  private ws?: WebSocket;
  private stream?: MediaStream;

  async start(config: ScreenShareConfig) {
    // 1. 获取屏幕流
    this.stream = await navigator.mediaDevices.getDisplayMedia({
      video: {
        width: { ideal: config.width },
        height: { ideal: config.height },
        frameRate: { ideal: config.frameRate }
      },
      audio: config.includeAudio
    });

    // 2. 创建 MediaRecorder
    this.mediaRecorder = new MediaRecorder(this.stream, {
      mimeType: 'video/webm;codecs=vp8,opus'
    });

    // 3. 连接到 RTMP 服务器（通过 HTTP 接口）
    const rtmpUrl = await this.getRtmpUrl();

    // 4. 开始推流
    this.mediaRecorder.ondataavailable = async (event) => {
      if (event.data.size > 0) {
        // 转换为 FLV 格式并推送
        const flvData = await this.convertToFLV(event.data);
        await this.sendToRTMP(flvData);
      }
    };

    this.mediaRecorder.start(1000); // 每 1 秒一个 chunk
  }

  stop() {
    this.mediaRecorder?.stop();
    this.ws?.close();
    this.stream?.getTracks().forEach(track => track.stop());
  }
}
```

**注**: 由于浏览器无法直接进行 RTMP 推流，需要考虑以下方案：

#### 方案 A: 使用 WebRTC-RTMP 网关
- 前端通过 WebRTC 将流推送到服务器
- 后端启动 WebRTC 接收器，转换为 RTMP 推流到 livelib
- 需要：新增 WebRTC 接收服务

#### 方案 B: 使用 MediaRecorder + HTTP 上传（推荐用于初步实现）
- 前端使用 MediaRecorder 录制
- 将 chunks 上传到服务器端点
- 服务器将流写入 livelib Channel
- 优点：简单，无需新增服务
- 缺点：延迟较高

#### 方案 C: 使用 WASM + FLV 编码器（最优）
- 使用 WASM 编译的 FLV 编码器（如 flv.js）
- 在浏览器端实时编码 FLV
- 通过 HTTP POST 推送到服务器
- 服务器直接转发到 livelib

**推荐**: 先实现方案 B（快速验证），后续优化为方案 C。

### 3. Cinema 页面集成

**文件**: [synctv-web/src/views/Cinema.vue](../synctv-web/src/views/Cinema.vue)

无需改动！屏幕共享作为影片源添加后，复用现有播放逻辑：

1. 用户在 MoviePush 组件中添加屏幕共享影片
2. 影片添加到播放列表
3. 切换到该影片时，自动通过现有的 `/api/room/movie/live/hls/list/:movieId.m3u8` 接口播放

### 4. 类型定义更新

**文件**: [synctv-web/src/types/Movie.ts](../synctv-web/src/types/Movie.ts)

```typescript
export interface BaseMovieInfo {
  // ... 现有字段
  vendorInfo?: VendorInfo;
}

export interface VendorInfo {
  vendor?: string;
  bilibili?: BilibiliStreamingInfo;
  alist?: AlistStreamingInfo;
  emby?: EmbyStreamingInfo;
  screenShareConfig?: ScreenShareConfig;  // 新增
}

export interface ScreenShareConfig {
  includeAudio: boolean;
  quality: 'high' | 'medium' | 'low';
  frameRate: number;
  bitrate: number;
}
```

---

## 详细任务分解

### 阶段 1：后端基础支持

#### 任务 1.1: 数据模型扩展
- [ ] 添加 `ScreenShareConfig` 结构体
- [ ] 在 `MovieBase` 中添加 `ScreenShareConfig` 字段
- [ ] 添加 `VendorScreenShare` 常量
- **预估**: 1 小时

#### 任务 1.2: 影片验证更新
- [ ] 在 `Validate()` 中添加屏幕共享影片验证
- **预估**: 1 小时

#### 任务 1.3: MovieInfo 生成更新
- [ ] 在 `genMovieInfo()` 中添加屏幕共享 URL 生成逻辑
- **预估**: 1 小时

**阶段 1 小计**: 3 小时

---

### 阶段 2：前端 UI 实现

#### 任务 2.1: MoviePush 组件扩展
- [ ] 添加 `SCREEN_SHARE` 枚举值
- [ ] 添加屏幕共享类型配置
- [ ] 添加质量选择 UI
- [ ] 添加音频开关 UI
- **预估**: 4 小时

#### 任务 2.2: 类型定义更新
- [ ] 更新 `BaseMovieInfo` 接口
- [ ] 添加 `ScreenShareConfig` 接口
- **预估**: 0.5 小时

**阶段 2 小计**: 4.5 小时

---

### 阶段 3：屏幕推流实现（方案 B：MediaRecorder + HTTP）

#### 任务 3.1: 创建 ScreenSharePublisher 组件
- [ ] 组件基础结构
- [ ] 屏幕捕获功能
- [ ] MediaRecorder 配置
- **预估**: 4 小时

#### 任务 3.2: FLV 格式转换（使用库）
- [ ] 引入 flv.js 或类似库
- [ ] 实现 WebM → FLV 转换
- **预估**: 6 小时

#### 任务 3.3: HTTP 推流接口
- [ ] 后端添加推流接收端点
- [ ] 前端实现 HTTP POST 推送
- [ ] 处理断线重连
- **预估**: 6 小时

#### 任务 3.4: 控制面板 UI
- [ ] 开始/停止共享按钮
- [ ] 共享状态显示
- [ ] 错误处理提示
- **预估**: 3 小时

**阶段 3 小计**: 19 小时

---

### 阶段 4：后端流处理

#### 任务 4.1: 推流接收 API
- [ ] 创建 `/api/room/movie/screen-share/push` 端点
- [ ] 认证和权限检查
- [ ] 获取或创建 livelib Channel
- **预估**: 4 小时

#### 任务 4.2: 流写入 Channel
- [ ] 将接收到的 FLV chunks 写入 Channel
- [ ] 处理流结束
- **预估**: 3 小时

**阶段 4 小计**: 7 小时

---

### 阶段 5：UI/UX 优化

#### 任务 5.1: 质量预设配置
- [ ] 高清 (1080p@30fps, 2Mbps)
- [ ] 标清 (720p@15fps, 1Mbps)
- [ ] 流畅 (480p@15fps, 500Kbps)
- **预估**: 2 小时

#### 任务 5.2: 错误处理完善
- [ ] 浏览器兼容性检测
- [ ] 权限拒绝处理
- [ ] 网络断开重连
- **预估**: 3 小时

#### 任务 5.3: 状态指示
- [ ] 共享中状态
- [ ] 推流连接状态
- [ ] 观看人数显示
- **预估**: 2 小时

**阶段 5 小计**: 7 小时

---

### 阶段 6：测试与优化

#### 任务 6.1: 功能测试
- [ ] 屏幕捕获测试
- [ ] 推流功能测试
- [ ] 播放功能测试
- **预估**: 4 小时

#### 任务 6.2: 性能优化
- [ ] 降低延迟
- [ ] 减少 CPU 占用
- [ ] 优化带宽使用
- **预估**: 6 小时

**阶段 6 小计**: 10 小时

---

## 总工作量估算

| 阶段 | 任务 | 预估工作量 |
|------|------|-----------|
| 阶段 1 | 后端基础支持 | 3 小时 |
| 阶段 2 | 前端 UI 实现 | 4.5 小时 |
| 阶段 3 | 屏幕推流实现 | 19 小时 |
| 阶段 4 | 后端流处理 | 7 小时 |
| 阶段 5 | UI/UX 优化 | 7 小时 |
| 阶段 6 | 测试与优化 | 10 小时 |
| **总计** | | **50.5 小时** |

---

## 技术难点与解决方案

### 难点 1: 浏览器无法直接推流 RTMP

**问题**: 浏览器没有原生的 RTMP 推流能力。

**解决方案**:
- **方案 B (当前)**: MediaRecorder → HTTP POST → 后端写入 Channel
- **方案 C (后续)**: 使用 WASM FLV 编码器实现更高效的推流

### 难点 2: FLV 格式编码

**问题**: 需要将 MediaRecorder 输出的 WebM 转换为 FLV 格式。

**解决方案**:
- 使用开源库 `flv.js` 或 `flv-encoder-js`
- 考虑在服务器端使用 FFmpeg 进行转码（需要服务器支持）

### 难点 3: 延迟控制

**问题**: 通过 HTTP 上传的延迟较高。

**解决方案**:
- 减小 chunk 大小（降低延迟）
- 使用 WebRTC 后期替换 HTTP 方案
- 优化服务器端流处理

---

## 验收测试清单

### 功能测试
- [ ] 可以选择屏幕/窗口/标签页共享
- [ ] 可以配置是否共享音频
- [ ] 可以选择视频质量（高清/标清/流畅）
- [ ] 添加影片后，房间内其他用户可以看到流
- [ ] 播放器正常播放共享内容
- [ ] 可以停止共享

### UI/UX 测试
- [ ] "屏幕共享"选项在添加影片中选择
- [ ] 质量选择器工作正常
- [ ] 音频开关工作正常
- [ ] 共享状态显示准确
- [ ] 错误提示友好

### 兼容性测试
- [ ] Chrome 最新版本
- [ ] Firefox 最新版本
- [ ] Edge 最新版本
- [ ] Safari（如支持 getDisplayMedia）

### 性能测试
- [ ] CPU 占用合理
- [ ] 延迟可接受（< 3秒）
- [ ] 带宽使用符合质量设置

---

## 相关文件

| 功能 | 文件路径 |
|------|----------|
| 后端数据模型 | `internal/model/movie.go` |
| 后端影片逻辑 | `internal/op/movie.go` |
| 后端 API 处理 | `server/handlers/movie.go` |
| 前端添加影片 | `synctv-web/src/components/cinema/MoviePush.vue` |
| 前端影片类型 | `synctv-web/src/types/Movie.ts` |
| 前端播放器 | `synctv-web/src/views/Cinema.vue` |
| RTMP 认证 | `internal/rtmp/rtmp.go` |
| livelib Channel | `internal/op/movie.go` (initChannel 方法) |

---

## 附录：后续优化方向

### 优化 1: WebRTC 替代 HTTP 推流
- 后端添加 WebRTC 接收服务
- 前端直接通过 WebRTC 推送流
- 降低延迟，提高稳定性

### 优化 2: WASM FLV 编码器
- 在浏览器端实时编码 FLV
- 减少服务器转码压力
- 提高编码效率

### 优化 3: 自适应码率
- 根据网络状况动态调整码率
- 保证流畅播放
