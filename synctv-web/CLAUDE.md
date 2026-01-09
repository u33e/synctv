# [根目录](../CLAUDE.md) > synctv-web (前端应用)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / synctv-web (前端应用)

## 模块概述

`synctv-web` 是 SyncTV 的前端应用，基于 Vue 3 + TypeScript + Vite 构建，提供用户友好的观影界面。

## 技术栈

| 技术 | 用途 |
|------|------|
| Vue 3 | 前端框架 |
| TypeScript | 类型安全 |
| Vite | 构建工具 |
| Pinia | 状态管理 |
| Vue Router | 路由 |
| Element Plus | UI 组件库 |
| TailwindCSS | CSS 框架 |
| ArtPlayer | 视频播放器 |
| HLS.js / DASH.js | 视频流处理 |
| Axios | HTTP 请求 |
| Animate.css | 动画效果 |

## 目录结构

```
synctv-web/
├── src/
│   ├── main.ts               # 入口文件
│   ├── App.vue              # 根组件
│   ├── router/               # 路由配置
│   │   └── index.ts
│   ├── stores/               # Pinia 状态管理
│   │   ├── index.ts
│   │   ├── user.ts          # 用户状态
│   │   └── room.ts          # 房间状态
│   ├── components/           # 组件
│   │   ├── cinema/           # 影院相关
│   │   ├── user/            # 用户相关
│   │   ├── admin/           # 管理相关
│   │   ├── fileList/        # 文件列表
│   │   ├── icons/           # 图标
│   │   ├── Player.vue       # 播放器组件
│   │   ├── RoomList.vue     # 房间列表
│   │   └── ...
│   ├── views/                # 页面
│   │   ├── Cinema.vue       # 影院页面
│   │   ├── HomeView.vue     # 首页
│   │   ├── CreateRoom.vue   # 创建房间
│   │   ├── JoinRoom.vue     # 加入房间
│   │   ├── Rooms.vue        # 房间列表
│   │   ├── SearchPage.vue   # 搜索页面
│   │   ├── auth/           # 认证页面
│   │   ├── admin/          # 管理后台
│   │   ├── user/           # 用户中心
│   │   └── error/          # 错误页面
│   ├── hooks/               # 组合式函数
│   │   ├── useRoom.ts       # 房间相关
│   │   ├── useScreen.ts     # 屏幕相关
│   │   ├── useSettings.ts   # 设置相关
│   │   └── useMovie.ts     # 影片相关
│   ├── services/            # API 服务
│   │   └── apis/
│   │       └── user.ts      # 用户 API
│   ├── types/               # 类型定义
│   │   ├── User.ts
│   │   └── Room.ts
│   ├── proto/               # Protobuf 定义
│   │   ├── message.proto
│   │   └── message.ts
│   ├── plugins/             # 插件
│   │   ├── artplayer-plugin-ass/  # 字幕插件
│   │   ├── control.ts            # 控制插件
│   │   ├── danmu.ts             # 弹幕插件
│   │   ├── source.ts            # 源插件
│   │   ├── subtitle.ts          # 字幕插件
│   │   └── sync.ts             # 同步插件
│   ├── utils/               # 工具函数
│   │   ├── index.ts
│   │   ├── requests.ts      # HTTP 请求封装
│   │   ├── notify.ts        # 通知
│   │   └── nprogress.ts    # 进度条
│   └── assets/             # 静态资源
│       ├── global.less      # 全局样式
│       ├── reset.css        # 重置样式
│       ├── animation.less   # 动画样式
│       └── appIcons/       # 应用图标
├── public/                 # 公共资源
│   ├── favicon.svg
│   ├── sw.js             # Service Worker
│   └── sw.media.js
├── package.json
├── vite.config.mts
├── tsconfig.json
└── tailwind.config.js
```

## 核心页面

### Cinema (影院页面)

**路径**: `/room/:id`

**功能**:
- 视频播放
- 弹幕显示
- 房间成员列表
- 影片列表管理
- 播放状态同步

### CreateRoom (创建房间)

**路径**: `/create`

**功能**:
- 创建新房间
- 设置房间密码
- 配置房间权限

### JoinRoom (加入房间)

**路径**: `/join`

**功能**:
- 通过 ID/名称加入房间
- 密码验证

### HomeView (首页)

**路径**: `/`

**功能**:
- 首页展示
- 快速导航

### Rooms (房间列表)

**路径**: `/rooms`

**功能**:
- 浏览所有房间
- 搜索房间

### SearchPage (搜索页面)

**路径**: `/search`

**功能**:
- 搜索影片
- 添加到房间

### Admin (管理后台)

**路径**: `/admin`

**功能**:
- 用户管理
- 房间管理
- 厂商管理
- 站点设置

## 状态管理

### user store

```typescript
export function userStore() {
  const token = ref<string>("");
  const userInfo = ref<User | null>(null);

  const login = async (data: LoginForm) => {};
  const logout = () => {};
  const getUserInfo = async () => {};
  // ...
}
```

### room store

```typescript
export function roomStore() {
  const currentRoom = ref<Room | null>(null);
  const members = ref<Member[]>([]);
  const movies = ref<Movie[]>([]);

  const joinRoom = async (roomId: string) => {};
  const leaveRoom = () => {};
  // ...
}
```

## Hooks (组合式函数)

### useRoom

房间相关逻辑，包括 WebSocket 连接、消息处理。

### useScreen

屏幕共享功能。

### useSettings

设置管理。

### useMovie

影片操作。

## 播放器插件

### artplayer-plugin-ass

ASS 字幕支持插件。

### control

播放器控制。

### danmu

弹幕功能。

### source

视频源切换。

### subtitle

字幕加载和切换。

### sync

播放同步。

## 路由配置

```typescript
const router = createRouter({
  routes: [
    { path: "/", component: HomeView },
    { path: "/create", component: CreateRoom },
    { path: "/join", component: JoinRoom },
    { path: "/rooms", component: Rooms },
    { path: "/room/:id", component: Cinema },
    { path: "/search", component: SearchPage },
    { path: "/auth/login", component: Login },
    { path: "/auth/register", component: Register },
    { path: "/admin", component: Admin },
    // ...
  ],
});
```

## 视频播放支持

| 协议 | 支持情况 | 实现方式 |
|------|---------|---------|
| HLS (.m3u8) | 支持 | HLS.js |
| DASH (.mpd) | 支持 | DASH.js |
| FLV | 支持 | mpegts.js |
| MP4/WebM | 支持 | 原生支持 |
| 直播流 | 支持 | HLS/FLV |

## 厂商集成

### Bilibili

- 直播播放
- 视频播放
- 弹幕支持

### Alist

- 文件列表浏览
- 视频播放

### Emby

- 媒体库浏览
- 视频播放
- 字幕支持

## 开发命令

```bash
# 安装依赖
pnpm install

# 开发服务器 (端口 8085)
pnpm dev

# 构建生产版本
pnpm build

# 预览构建
pnpm preview

# 代码检查
pnpm lint

# 格式化代码
pnpm format

# 类型检查
pnpm type-check
```

## 构建输出

构建产物输出到 `dist/` 目录，后端通过 `embed` 指令嵌入到 Go 二进制文件中。

## Service Worker

支持 PWA 功能:
- 离线缓存
- 媒体缓存

## Protobuf 集成

使用 `ts-proto` 生成 TypeScript 代码:

```bash
./proto.sh
```

## 常见问题 (FAQ)

### 如何修改默认端口?

修改 `package.json` 中的 dev 脚本:

```json
"dev": "vite --port 3000 --strictPort"
```

### 如何配置后端地址?

通过环境变量或直接修改 `utils/requests.ts`。

### 如何添加新的视频格式?

在播放器组件中添加对应的解码器支持。

### 如何部署?

```bash
pnpm build
```

然后将 `dist/` 目录内容部署到静态服务器。

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [src/main.ts](src/main.ts) | 入口文件 |
| [src/App.vue](src/App.vue) | 根组件 |
| [src/router/index.ts](src/router/index.ts) | 路由配置 |
| [src/stores/user.ts](src/stores/user.ts) | 用户状态 |
| [src/stores/room.ts](src/stores/room.ts) | 房间状态 |
| [src/views/Cinema.vue](src/views/Cinema.vue) | 影院页面 |
| [src/components/Player.vue](src/components/Player.vue) | 播放器 |
| [package.json](package.json) | 项目配置 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
