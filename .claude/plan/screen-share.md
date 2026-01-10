# SyncTV 屏幕共享功能实施规划

> 规划版本: v1.0
> 创建日期: 2026-01-10
> 状态: 待确认

---

## 已明确的决策

### 技术选型
- **屏幕捕获 API**: 使用 `navigator.mediaDevices.getDisplayMedia()` 原生浏览器 API
- **视频传输协议**: 复用现有 WebRTC 基础设施（无需后端改动）
- **信令机制**: 复用现有 WebSocket 信令消息类型（WEBRTC_OFFER/ANSWER/ICE_CANDIDATE/JOIN/LEAVE）
- **权限控制**: 复用现有 `PermissionWebRTC` 权限
- **UI 框架**: Element Plus + TailwindCSS（与现有前端技术栈一致）

### 架构决策
- **后端改动**: 无需改动（现有信令机制已支持多轨道）
- **前端改动**: 主要集中在 `synctv-web/src/views/Cinema.vue`
- **协议改动**: 无需改动现有 Protobuf 定义

---

## 整体规划概述

### 项目目标

实现 SyncTV 房间内的屏幕共享功能，允许用户：
1. 通过浏览器原生 API 共享整个屏幕、应用窗口或浏览器标签页
2. 共享时同时传输音频（如需要）
3. 房间内其他用户可实时观看共享的屏幕内容
4. 支持同时显示多个共享屏幕（类似现有多音频功能）
5. 提供流畅的开启/停止控制体验

### 技术栈

| 类别 | 技术 |
|------|------|
| 前端框架 | Vue 3 + TypeScript + Vite |
| UI 组件库 | Element Plus + TailwindCSS |
| WebRTC | RTCPeerConnection (现有) |
| 屏幕捕获 | navigator.mediaDevices.getDisplayMedia() |
| 信令协议 | WebSocket (现有) |
| 视频显示 | HTML5 Video 元素 |

### 主要阶段

1. **阶段 1**: 核心功能实现 - 屏幕捕获与视频轨道传输
2. **阶段 2**: UI/UX 优化 - 多视频布局与交互改进
3. **阶段 3**: 兼容性与性能优化 - 浏览器适配与带宽控制

---

## 详细任务分解

### 阶段 1：核心功能实现

#### 任务 1.1：屏幕捕获功能实现

**目标**: 实现屏幕/窗口/标签页的选择与流获取

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
// 新增状态变量
const screenStream = ref<MediaStream | undefined>(undefined);
const isScreenSharing = ref(false);

// 新增屏幕捕获函数
const startScreenShare = async (includeAudio = false) => {
  try {
    const displayMediaOptions: DisplayMediaStreamOptions = {
      video: {
        displaySurface: "monitor", // 可选: monitor/window/browser
        cursor: "always"
      },
      audio: includeAudio ? {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true
      } : false
    };

    screenStream.value = await navigator.mediaDevices.getDisplayMedia(displayMediaOptions);
    isScreenSharing.value = true;

    // 监听流结束事件（用户停止共享）
    screenStream.value.getVideoTracks()[0].addEventListener('ended', () => {
      stopScreenShare();
    });

    return true;
  } catch (err) {
    ElMessage.error(`屏幕共享失败: ${err}`);
    return false;
  }
};
```

**预估工作量**: 2 小时

---

#### 任务 1.2：视频轨道添加到 PeerConnection

**目标**: 将屏幕捕获的视频轨道添加到现有 WebRTC 连接

**涉及文件**:
- `synctv-web/src/views/Cinema.vue` - 修改 `createPeerConnection` 函数

**实施内容**:
```typescript
const createPeerConnection = (id: string) => {
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
    iceCandidatePoolSize: 10
  });

  pc.onicecandidate = (event) => {
    if (event.candidate) {
      sendElement(
        Message.create({
          type: MessageType.WEBRTC_ICE_CANDIDATE,
          webrtcData: {
            data: JSON.stringify(event.candidate),
            to: id
          }
        })
      );
    }
  };

  pc.ontrack = (event) => {
    // 区分音频和视频轨道
    if (event.track.kind === 'video') {
      handleVideoTrack(event.streams[0], id, event.track);
    } else if (event.track.kind === 'audio') {
      handleAudioTrack(event.streams[0], id);
    }
  };

  // 添加本地音频流（麦克风）
  if (localStream.value) {
    localStream.value.getAudioTracks().forEach((track) => pc.addTrack(track, localStream.value!));
  }

  // 添加屏幕共享流（视频 + 可选音频）
  if (screenStream.value) {
    screenStream.value.getTracks().forEach((track) => pc.addTrack(track, screenStream.value!));
  }

  peerConnections.value[id] = pc;
  return pc;
};

// 处理视频轨道
const handleVideoTrack = (stream: MediaStream, id: string, track: MediaStreamTrack) => {
  // 创建或更新视频元素
  let videoElement = remoteVideoElements[id];
  if (!videoElement) {
    videoElement = document.createElement("video");
    videoElement.autoplay = true;
    videoElement.playsInline = true;
    videoElement.muted = false; // 允许播放共享内容的音频
    videoElement.className = "remote-video";
    videoElement.onended = () => {
      delete remoteVideoElements[id];
    };
    remoteVideoElements[id] = videoElement;
  }

  videoElement.srcObject = stream;
  updateVideoLayout(); // 更新视频布局
};
```

**预估工作量**: 3 小时

---

#### 任务 1.3：停止屏幕共享功能

**目标**: 实现停止屏幕共享的清理逻辑

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
const stopScreenShare = () => {
  if (!screenStream.value) return;

  // 停止所有轨道
  screenStream.value.getTracks().forEach((track) => {
    track.stop();
  });

  // 从所有 PeerConnection 中移除视频轨道的发送者
  for (const [id, pc] of Object.entries(peerConnections.value)) {
    const senders = pc.getSenders();
    const videoSenders = senders.filter((s) => s.track?.kind === 'video');
    videoSenders.forEach((sender) => pc.removeTrack(sender));
  }

  // 清理状态
  screenStream.value = undefined;
  isScreenSharing.value = false;

  // 重新协商（发送新的 offer）
  renegotiateAllConnections();

  ElMessage.success("已停止屏幕共享");
};

// 重新协商所有连接
const renegotiateAllConnections = async () => {
  for (const [id, pc] of Object.entries(peerConnections.value)) {
    try {
      const offer = await pc.createOffer();
      await pc.setLocalDescription(offer);
      sendElement(
        Message.create({
          type: MessageType.WEBRTC_OFFER,
          webrtcData: {
            data: JSON.stringify(offer),
            to: id
          }
        })
      );
    } catch (err) {
      console.error(`Renegotiation failed for ${id}:`, err);
    }
  }
};
```

**预估工作量**: 2 小时

---

#### 任务 1.4：远端视频元素管理

**目标**: 建立远端视频元素的存储与管理机制

**涉及文件**:
- `synctv-web/src/views/Cinema.vue` - 新增 `remoteVideoElements` 变量

**实施内容**:
```typescript
// 新增状态变量
let remoteVideoElements: { [key: string]: HTMLVideoElement } = {};

// 视频容器引用
const videoContainer = ref<HTMLElement | null>(null);

// 清理函数（在 exitWebRTC 中调用）
const cleanupRemoteVideos = () => {
  for (const [id, video] of Object.entries(remoteVideoElements)) {
    video.pause();
    video.srcObject = null;
    delete remoteVideoElements[id];
  }
};
```

**预估工作量**: 1 小时

---

### 阶段 2：UI/UX 优化

#### 任务 2.1：屏幕共享控制面板 UI

**目标**: 设计并实现屏幕共享控制界面

**UI 设计需求**:
- 开启屏幕共享按钮（位置与现有"加入语音"按钮并列）
- 停止屏幕共享按钮（开启后显示）
- 音频选项（是否共享系统音频）
- 共享状态指示器

**涉及文件**:
- `synctv-web/src/views/Cinema.vue` - template 部分添加 UI

**实施内容**:
```vue
<!-- 在音频控制面板下方添加屏幕共享控制面板 -->
<div
  ref="screenShareControls"
  v-show="screenStream"
  class="card-body mb-2 screen-share-controls-container"
>
  <div class="screen-share-controls">
    <div class="screen-share-status">
      <el-tag type="success" effect="dark">
        <el-icon><Monitor /></el-icon>
        正在共享屏幕 ({{ Object.keys(remoteVideoElements).length }} 人观看)
      </el-tag>
    </div>

    <el-switch
      v-model="shareSystemAudio"
      active-text="共享系统音频"
      @change="toggleSystemAudio"
    />

    <el-button
      @click="stopScreenShare"
      type="danger"
      size="small"
    >
      停止共享
    </el-button>
  </div>
</div>

<!-- 开启按钮（与加入语音并列） -->
<el-button
  v-if="!screenStream && can(RoomMemberPermission.PermissionWebRTC)"
  @click="openScreenShareDialog"
  type="success"
  size="small"
  style="float: right"
>
  <el-icon><Monitor /></el-icon>
  屏幕共享
</el-button>
```

**预估工作量**: 4 小时（含样式调整）

---

#### 任务 2.2：多视频网格布局

**目标**: 实现适配不同数量视频的网格布局

**UI 设计需求**:
- 1 个视频: 全屏或大尺寸显示
- 2 个视频: 左右分屏
- 3-4 个视频: 2x2 网格
- 5+ 个视频: 3x2 或滚动网格
- 视频应保持宽高比（16:9 或自适应）
- 支持拖拽调整视频顺序

**涉及文件**:
- `synctv-web/src/views/Cinema.vue` - template 添加视频容器，style 添加网格样式

**实施内容**:
```vue
<!-- 视频容器 -->
<div
  ref="videoContainer"
  v-show="Object.keys(remoteVideoElements).length > 0"
  class="video-grid-container"
>
  <div
    v-for="(video, userId) in remoteVideoElements"
    :key="userId"
    class="video-wrapper"
  >
    <video
      ref="video"
      :id="`video-${userId}`"
      class="remote-video"
    ></video>
    <div class="video-label">
      <el-tag size="small">{{ getUserLabel(userId) }}</el-tag>
    </div>
  </div>
</div>
```

```less
<style lang="less" scoped>
// 视频网格布局
.video-grid-container {
  position: fixed;
  top: 20px;
  right: 20px;
  width: 320px;
  max-height: 60vh;
  background: rgba(0, 0, 0, 0.8);
  border-radius: 8px;
  padding: 10px;
  display: grid;
  gap: 8px;
  overflow-y: auto;
  z-index: 100;
  transition: all 0.3s ease;

  // 根据视频数量动态调整网格
  &:has(.video-wrapper:nth-child(1):last-child) {
    grid-template-columns: 1fr;
  }

  &:has(.video-wrapper:nth-child(2):last-child) {
    grid-template-columns: 1fr;
  }

  &:has(.video-wrapper:nth-child(3):last-child),
  &:has(.video-wrapper:nth-child(4):last-child) {
    grid-template-columns: 1fr 1fr;
  }

  &:has(.video-wrapper:nth-child(n+5)) {
    grid-template-columns: 1fr 1fr 1fr;
  }
}

.video-wrapper {
  position: relative;
  aspect-ratio: 16/9;
  background: #000;
  border-radius: 4px;
  overflow: hidden;

  .remote-video {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .video-label {
    position: absolute;
    top: 5px;
    left: 5px;
    z-index: 10;
  }
}

// 屏幕共享控制面板样式
.screen-share-controls-container {
  transition: all 0.3s ease-in-out;
  transform-origin: top;
  animation: slideDown 0.3s ease-in-out;
}

.screen-share-controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 8px;

  .screen-share-status {
    text-align: center;
  }
}
</style>
```

**预估工作量**: 6 小时

---

#### 任务 2.3：全屏查看共享屏幕功能

**目标**: 允许用户点击视频进入全屏模式

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
const toggleFullScreen = (element: HTMLElement) => {
  if (!document.fullscreenElement) {
    element.requestFullscreen().catch((err) => {
      ElMessage.error(`全屏失败: ${err.message}`);
    });
  } else {
    document.exitFullscreen();
  }
};

// 在视频元素上添加点击事件
video.onclick = () => toggleFullScreen(video);
```

**预估工作量**: 1 小时

---

### 阶段 3：兼容性与性能优化

#### 任务 3.1：浏览器兼容性处理

**目标**: 确保功能在各主流浏览器上正常工作

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
const checkBrowserSupport = () => {
  // 检查 getDisplayMedia 支持
  if (!navigator.mediaDevices || !navigator.mediaDevices.getDisplayMedia) {
    ElNotification({
      title: "浏览器不支持",
      message: "您的浏览器不支持屏幕共享功能，请使用最新版本的 Chrome、Firefox 或 Edge",
      type: "warning",
      duration: 5000
    });
    return false;
  }
  return true;
};

// 在开启屏幕共享前调用
const openScreenShareDialog = async () => {
  if (!checkBrowserSupport()) return;
  // ... 继续原有逻辑
};
```

**预估工作量**: 2 小时

---

#### 任务 3.2：视频编码参数优化

**目标**: 根据网络条件动态调整视频质量和码率

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
const getVideoConstraints = () => {
  // 根据设备性能和用户偏好选择合适参数
  const constraints: DisplayMediaStreamConstraints = {
    video: {
      // 分辨率限制
      width: { ideal: 1920, max: 1920 },
      height: { ideal: 1080, max: 1080 },
      // 帧率限制
      frameRate: { ideal: 30, max: 30 },
      // 显示表面类型
      displaySurface: "monitor",
      cursor: "always"
    },
    audio: shareSystemAudio.value
  };

  return constraints;
};

// 在 PeerConnection 中设置编码参数
const setVideoEncodingParameters = (pc: RTCPeerConnection) => {
  const senders = pc.getSenders();
  senders.forEach((sender) => {
    if (sender.track?.kind === 'video') {
      const params = sender.getParameters();
      if (!params.encodings) {
        params.encodings = [{}];
      }

      // 设置码率限制 (kbps)
      params.encodings[0].maxBitrate = 2000;
      params.encodings[0].maxFramerate = 30;

      sender.setParameters(params).catch((err) => {
        console.error('Failed to set encoding parameters:', err);
      });
    }
  });
};
```

**预估工作量**: 3 小时

---

#### 任务 3.3：网络状态监测与自适应

**目标**: 监测网络状态并根据情况调整传输参数

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
const monitorConnectionQuality = (pc: RTCPeerConnection) => {
  pc.addEventListener('connectionstatechange', () => {
    const state = pc.connectionState;
    console.log(`Connection state changed to: ${state}`);

    switch (state) {
      case 'connected':
        ElMessage.success('视频连接已建立');
        break;
      case 'disconnected':
        ElMessage.warning('视频连接已断开，尝试重新连接...');
        break;
      case 'failed':
        ElMessage.error('视频连接失败');
        break;
    }
  });

  // 监听 ICE 连接状态
  pc.addEventListener('iceconnectionstatechange', () => {
    const state = pc.iceConnectionState;
    console.log(`ICE connection state: ${state}`);

    if (state === 'failed') {
      // ICE 连接失败，尝试重启 ICE
      pc.restartIce();
    }
  });
};
```

**预估工作量**: 2 小时

---

#### 任务 3.4：权限和错误处理完善

**目标**: 完善各种边界情况的错误处理

**涉及文件**:
- `synctv-web/src/views/Cinema.vue`

**实施内容**:
```typescript
const handleScreenShareError = (error: any) => {
  console.error('Screen share error:', error);

  let message = '屏幕共享失败';

  if (error.name === 'NotAllowedError') {
    message = '您取消了屏幕共享选择';
  } else if (error.name === 'NotFoundError') {
    message = '未找到可用的屏幕或窗口';
  } else if (error.name === 'NotReadableError') {
    message = '无法读取屏幕内容，请尝试其他屏幕或窗口';
  } else if (error.name === 'InvalidStateError') {
    message = '屏幕共享已处于活动状态';
  } else {
    message = `屏幕共享失败: ${error.message}`;
  }

  ElNotification({
    title: '屏幕共享错误',
    message,
    type: 'error'
  });

  // 重置状态
  isScreenSharing.value = false;
  if (screenStream.value) {
    screenStream.value.getTracks().forEach(t => t.stop());
    screenStream.value = undefined;
  }
};
```

**预估工作量**: 2 小时

---

## 需要进一步明确的问题

### 问题 1：视频显示位置与布局方案

**推荐方案**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **方案 A: 浮动面板布局** | 视频以浮动面板形式显示在屏幕右上角，用户可拖动调整位置 | 不遮挡主视频播放器，灵活性强 | 可能与播放器控件冲突，移动端体验一般 |
| **方案 B: 侧边栏布局** | 在右侧聊天面板下方或侧边栏中显示共享视频 | 布局清晰，易于管理多个视频 | 视频尺寸可能较小 |
| **方案 C: 模态窗口布局** | 以模态窗口形式展示共享内容，用户可点击打开/关闭 | 不占用页面空间，不影响主要功能 | 需要用户主动点击才能查看 |

**等待用户选择**：

```
请选择您偏好的方案，或提供其他建议：
[ ] 方案 A - 浮动面板布局
[ ] 方案 B - 侧边栏布局
[ ] 方案 C - 模态窗口布局
[ ] 其他方案：__________________
```

---

### 问题 2：屏幕共享音频处理策略

**推荐方案**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **方案 A: 默认不共享音频，用户可选择开启** | 屏幕共享默认仅传输视频，用户可通过开关选择是否共享系统音频 | 减少带宽占用，避免隐私泄露 | 需要用户手动操作 |
| **方案 B: 始终提示用户是否共享音频** | 每次开启屏幕共享时都弹出选项让用户选择是否共享音频 | 给用户明确选择权 | 操作步骤增加 |
| **方案 C: 根据房间设置决定** | 房间创建者可在设置中配置"屏幕共享默认是否包含音频" | 灵活适应不同场景需求 | 需要后端支持新配置项 |

**等待用户选择**：

```
请选择您偏好的方案，或提供其他建议：
[ ] 方案 A - 默认不共享音频，用户可选择开启
[ ] 方案 B - 始终提示用户是否共享音频
[ ] 方案 C - 根据房间设置决定
[ ] 其他方案：__________________
```

---

### 问题 3：多屏幕共享场景下的处理方式

**推荐方案**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **方案 A: 允许多人同时共享，网格显示** | 房间内多个用户可同时共享屏幕，视频以网格布局同时显示 | 支持多人协作场景 | 网络和带宽压力较大 |
| **方案 B: 同一时间只能一人共享，采用"抢麦"机制** | 当前共享者停止后，其他人才能开始共享 | 带宽压力小，观看体验好 | 无法支持多人协作 |
| **方案 C: 混合模式 - 设置"主讲人"角色** | 设置"主讲人"权限，只有主讲人可以共享屏幕，其他人可举手申请 | 适合会议/教学场景 | 需要新增角色权限系统 |

**等待用户选择**：

```
请选择您偏好的方案，或提供其他建议：
[ ] 方案 A - 允许多人同时共享，网格显示
[ ] 方案 B - 同一时间只能一人共享，采用"抢麦"机制
[ ] 方案 C - 混合模式 - 设置"主讲人"角色
[ ] 其他方案：__________________
```

---

### 问题 4：视频质量与带宽控制策略

**推荐方案**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **方案 A: 固定高质量 (1080p@30fps, 2Mbps)** | 使用较高的固定码率和分辨率 | 视频质量好，适合高速网络 | 对网络要求高，低带宽环境下卡顿 |
| **方案 B: 自适应质量 (根据网络动态调整)** | 监测网络状态，在 360p-1080p 和 15-30fps 之间动态调整 | 适应各种网络环境 | 质量波动可能影响体验 |
| **方案 C: 用户手动选择** | 提供清晰度选项（高清/标清/流畅），用户根据网络情况选择 | 用户有完全控制权 | 增加用户操作复杂度 |

**等待用户选择**：

```
请选择您偏好的方案，或提供其他建议：
[ ] 方案 A - 固定高质量 (1080p@30fps, 2Mbps)
[ ] 方案 B - 自适应质量 (根据网络动态调整)
[ ] 方案 C - 用户手动选择
[ ] 其他方案：__________________
```

---

## 用户反馈区域

请在此区域补充您对整体规划的意见和建议：

```
用户补充内容：




```

---

## 验收测试清单

### 功能测试

- [ ] 用户可以成功开启屏幕共享
- [ ] 可以选择共享整个屏幕/窗口/标签页
- [ ] 房间内其他用户可以看到共享的视频
- [ ] 视频播放流畅，无明显卡顿
- [ ] 音频可以正常共享（如果启用）
- [ ] 用户可以成功停止屏幕共享
- [ ] 当用户停止共享时，远端视频正确消失
- [ ] 网络断开时，有适当的错误提示

### UI/UX 测试

- [ ] 开启/停止按钮显示正确
- [ ] 共享状态指示准确
- [ ] 多视频布局在不同数量下显示正常
- [ ] 视频保持正确的宽高比
- [ ] 全屏功能正常工作
- [ ] 移动端布局适配良好

### 兼容性测试

- [ ] Chrome 最新版本测试通过
- [ ] Firefox 最新版本测试通过
- [ ] Edge 最新版本测试通过
- [ ] Safari 测试（如支持）
- [ ] 不支持的浏览器有友好提示

### 性能测试

- [ ] 单人共享时 CPU 占用正常
- [ ] 多人共享时网络带宽可接受
- [ ] 视频延迟低于 500ms
- [ ] 内存占用在合理范围内

### 边界情况测试

- [ ] 用户拒绝共享权限时的处理
- [ ] 共享过程中窗口关闭时的处理
- [ ] 快速开启/停止共享时的稳定性
- [ ] 房间内多人同时开启共享时的行为
- [ ] 弱网环境下的降级表现

---

## 工作量估算

| 阶段 | 任务 | 预估工作量 |
|------|------|-----------|
| 阶段 1 | 任务 1.1: 屏幕捕获功能实现 | 2 小时 |
| 阶段 1 | 任务 1.2: 视频轨道添加到 PeerConnection | 3 小时 |
| 阶段 1 | 任务 1.3: 停止屏幕共享功能 | 2 小时 |
| 阶段 1 | 任务 1.4: 远端视频元素管理 | 1 小时 |
| 阶段 2 | 任务 2.1: 屏幕共享控制面板 UI | 4 小时 |
| 阶段 2 | 任务 2.2: 多视频网格布局 | 6 小时 |
| 阶段 2 | 任务 2.3: 全屏查看共享屏幕功能 | 1 小时 |
| 阶段 3 | 任务 3.1: 浏览器兼容性处理 | 2 小时 |
| 阶段 3 | 任务 3.2: 视频编码参数优化 | 3 小时 |
| 阶段 3 | 任务 3.3: 网络状态监测与自适应 | 2 小时 |
| 阶段 3 | 任务 3.4: 权限和错误处理完善 | 2 小时 |
| **总计** | | **28 小时** |

---

## 附录：现有 WebRTC 代码参考

### 后端文件位置
- 信令处理: [server/handlers/websocket.go](../server/handlers/websocket.go)

### 前端文件位置
- 现有 WebRTC 实现: [synctv-web/src/views/Cinema.vue](../synctv-web/src/views/Cinema.vue)

### 权限定义
- 权限类型: [synctv-web/src/types/Room.ts](../synctv-web/src/types/Room.ts)
- 权限 Hook: [synctv-web/src/hooks/useRoom.ts](../synctv-web/src/hooks/useRoom.ts)
