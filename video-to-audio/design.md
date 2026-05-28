# 音视频转音频工具站 技术设计

## 1. 功能概述

为 3-5 人小团队提供一个私有音视频转音频工具站。用户登录后可输入视频 URL（支持 B站等平台）或上传本地视频文件，后端通过 yt-dlp + ffmpeg 转码为 mp3/aac/wav，前端实时显示转码进度，转换完成后支持在线预览和下载，并保留每账号最近 30 条历史记录。

---

## 2. 涉及模块

| 模块 | 路径 | 说明 |
|------|------|------|
| 前端 | `/Users/meituan/Desktop/video-to-audio/` | React 19 + Webpack 5 + TypeScript |
| 后端 | `/Users/meituan/Desktop/video-to-audio-backend/` | Node.js + Express + TypeScript + SQLite |
| infra | `/Users/meituan/Desktop/infra/video-to-audio/` | Docker Compose + Nginx，复用现有体系 |

---

## 3. 页面设计

### 主页（唯一页面，路由 `/`）

未登录时重定向到 `/login`，登录后展示主页。

#### 功能描述

页面分为上下两个区域：
- **上方：转换操作区** —— 输入 URL 或上传文件、选择输出格式、触发转换、实时进度、预览+下载
- **下方：历史记录区** —— 当前用户最近 30 条转换记录列表，支持重新下载/预览/删除

#### 交互流程

**URL 转换流程：**
- When 用户输入视频 URL 并点击"开始转换"，the system shall 调用后端 `/api/convert/url` 接口，建立 SSE 连接实时推送进度
- When SSE 推送 `done` 事件，the system shall 显示预览播放器和下载按钮，并刷新历史记录列表
- When SSE 推送 `error` 事件，the system shall 显示错误提示信息，转换状态重置为初始态

**文件上传转换流程：**
- When 用户选择本地文件，the system shall 将文件切分为 5MB 分片，逐片上传，显示上传进度条
- When 所有分片上传完成，the system shall 自动触发合并请求，后端合并后开始转码
- When 转码开始，the system shall 切换到 SSE 进度模式，实时展示转码进度
- When 转码完成，the system shall 显示预览播放器和下载按钮

**历史记录操作：**
- When 用户点击历史记录中的"预览"，the system shall 打开音频播放器播放该条记录的音频
- When 用户点击"下载"，the system shall 触发浏览器文件下载
- When 用户点击"删除"，the system shall 弹出确认框，确认后删除记录和服务器文件

#### 组件结构

```
src/
├── pages/
│   ├── Login/
│   │   └── index.tsx          # 登录页
│   └── Home/
│       ├── index.tsx          # 主页容器，负责布局
│       ├── components/
│       │   ├── ConvertPanel/
│       │   │   ├── index.tsx          # 转换操作区容器
│       │   │   ├── UrlInput.tsx       # URL 输入 + 格式选择 + 提交按钮
│       │   │   ├── FileUpload.tsx     # 拖拽/点击上传，分片逻辑
│       │   │   ├── ProgressBar.tsx    # 进度条（上传进度/转码进度复用）
│       │   │   └── ResultPanel.tsx    # 转换结果：预览播放器 + 下载按钮
│       │   └── HistoryList/
│       │       ├── index.tsx          # 历史记录列表容器
│       │       └── HistoryItem.tsx    # 单条记录（预览/下载/删除）
```

#### 状态管理

使用 React 本地状态（`useState` + `useReducer`）+ Context，无需引入额外状态管理库：

| 状态 | 位置 | 说明 |
|------|------|------|
| `convertState` | `ConvertPanel` 组件内 useReducer | 转换任务状态机（idle/uploading/converting/done/error） |
| `uploadProgress` | `FileUpload` 组件内 useState | 分片上传进度 0-100 |
| `convertProgress` | `ConvertPanel` 组件内 useState | SSE 推送的转码进度 0-100 |
| `result` | `ConvertPanel` 组件内 useState | 转换结果（文件 ID、预览 URL） |
| `historyList` | `HistoryList` 组件内 useState | 历史记录数组 |
| `userInfo` | `AuthContext`（全局） | 登录用户信息 + token |

---

## 4. 接口设计

### 4.1 登录接口（复用 security-quiz-game 模式）

#### POST `/api/user/login`
- **说明**：用户名+密码登录，返回 JWT AccessToken，RefreshToken 写入 HttpOnly Cookie
- **请求体**：
```typescript
interface LoginRequest {
  account: string;   // 用户名
  password: string;
}
```
- **响应**：
```typescript
interface LoginResponse {
  code: 200;
  data: {
    userInfo: { id: number; username: string; nickname: string };
    accessToken: string;
  };
}
```

#### POST `/api/user/refresh`
- **说明**：用 HttpOnly Cookie 中的 RefreshToken 换新 AccessToken

#### POST `/api/user/logout`
- **说明**：清除 RefreshToken Cookie

---

### 4.2 转换接口

#### POST `/api/convert/url`
- **说明**：提交 URL 转换任务，返回 `taskId`，随后通过 SSE 接口订阅进度
- **请求体**：
```typescript
interface ConvertUrlRequest {
  url: string;           // 视频 URL（直链或平台链接）
  format: 'mp3' | 'aac' | 'wav';
}
```
- **响应**：
```typescript
interface ConvertUrlResponse {
  code: 200;
  data: { taskId: string };
}
```

#### POST `/api/convert/upload/init`
- **说明**：初始化分片上传，返回 `uploadId`
- **请求体**：
```typescript
interface UploadInitRequest {
  filename: string;      // 原始文件名
  totalChunks: number;   // 总分片数
  format: 'mp3' | 'aac' | 'wav';
}
```
- **响应**：
```typescript
interface UploadInitResponse {
  code: 200;
  data: { uploadId: string };
}
```

#### POST `/api/convert/upload/chunk`
- **说明**：上传单个分片，multipart/form-data
- **Form 字段**：`uploadId`, `chunkIndex`（当前片序号，从0开始）, `chunk`（二进制数据）
- **响应**：`{ code: 200, data: { received: number } }`（已收到分片数）

#### POST `/api/convert/upload/merge`
- **说明**：所有分片上传完成后触发合并 + 转码，返回 `taskId`
- **请求体**：
```typescript
interface MergeRequest {
  uploadId: string;
}
```
- **响应**：
```typescript
interface MergeResponse {
  code: 200;
  data: { taskId: string };
}
```

#### GET `/api/convert/progress/:taskId`
- **说明**：SSE 接口，实时推送转码进度
- **响应头**：`Content-Type: text/event-stream`
- **事件格式**：
```typescript
// 进度更新
{ event: 'progress', data: { percent: number; stage: 'downloading' | 'converting' } }
// 完成
{ event: 'done', data: { fileId: string } }
// 错误
{ event: 'error', data: { message: string } }
```

---

### 4.3 文件接口

#### GET `/api/file/:fileId/preview`
- **说明**：流式返回音频文件，支持 Range 请求（在线播放必需）
- **响应头**：`Content-Type: audio/mpeg`（或对应格式），`Accept-Ranges: bytes`

#### GET `/api/file/:fileId/download`
- **说明**：触发浏览器下载
- **响应头**：`Content-Disposition: attachment; filename="xxx.mp3"`

---

### 4.4 历史记录接口

#### GET `/api/history`
- **说明**：获取当前用户最近 30 条记录（JWT 鉴权）
- **响应**：
```typescript
interface HistoryItem {
  id: number;
  fileId: string;
  originalName: string;   // 原始文件名或 URL
  format: string;         // 输出格式
  status: 'done' | 'error';
  fileSize: number;       // 字节
  duration: number;       // 秒
  createdAt: string;      // ISO 时间字符串
}
```

#### DELETE `/api/history/:id`
- **说明**：删除指定历史记录及服务器文件

---

## 5. 类型定义

| 类型 | 文件位置 | 说明 |
|------|---------|------|
| `ConvertState` | `src/types/convert.ts` | 前端转换状态机类型 |
| `HistoryItem` | `src/types/history.ts` | 历史记录条目类型 |
| `SSEEvent` | `src/types/sse.ts` | SSE 事件类型 |
| `TaskRow` | 后端 `src/database/task/index.ts` | 数据库任务行类型 |
| `HistoryRow` | 后端 `src/database/history/index.ts` | 数据库历史行类型 |

---

## 6. 数据库设计（SQLite）

### users 表（直接复用 security-quiz-game 结构）
```sql
-- 初期硬编码账号：启动时 seed 写入，存在则跳过
-- username: admin / password: 见环境变量 PRESET_PASSWORD
```

### tasks 表
```sql
CREATE TABLE IF NOT EXISTS tasks (
  id          TEXT PRIMARY KEY,          -- UUID，即 taskId
  user_id     INTEGER NOT NULL,
  type        TEXT NOT NULL,             -- 'url' | 'upload'
  source      TEXT NOT NULL,             -- URL 或原始文件名
  format      TEXT NOT NULL,             -- 'mp3' | 'aac' | 'wav'
  status      TEXT DEFAULT 'pending',   -- 'pending'|'processing'|'done'|'error'
  file_id     TEXT,                      -- 转换成功后的文件 ID
  error_msg   TEXT,
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### history 表
```sql
CREATE TABLE IF NOT EXISTS history (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id       INTEGER NOT NULL,
  task_id       TEXT NOT NULL,
  file_id       TEXT NOT NULL,
  original_name TEXT NOT NULL,
  format        TEXT NOT NULL,
  file_size     INTEGER,                -- 字节
  duration      REAL,                   -- 秒
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**超限策略**：每次写入 history 后，检查该用户记录数，超过 30 条则删除最旧的记录及对应文件。

---

## 7. 后端核心模块设计

### 转换任务流程

```
[前端] POST /api/convert/url  →  创建 task 记录（pending）
                              →  返回 taskId
[前端] GET  /api/convert/progress/:taskId  →  建立 SSE 连接

[后端] 异步执行：
  1. yt-dlp 下载视频到临时目录 /tmp/vta/{taskId}/source.*
     → SSE 推送 { stage: 'downloading', percent: 0~50 }
  2. ffmpeg 转码输出到 /app/files/{fileId}.{format}
     → SSE 推送 { stage: 'converting', percent: 50~100 }
  3. 写入 history 表，更新 task 状态为 done
     → SSE 推送 { event: 'done', data: { fileId } }
  4. 清理临时文件 /tmp/vta/{taskId}/
```

### 分片上传流程

```
[前端] POST /upload/init  →  创建 uploadId，初始化 /tmp/vta/uploads/{uploadId}/ 目录
[前端] POST /upload/chunk（并发3片）→  写入 /tmp/vta/uploads/{uploadId}/{chunkIndex}.part
[前端] POST /upload/merge  →  后端合并所有 .part 文件为完整视频
                           →  创建 task，走与 URL 转换相同的转码流程
```

---

## 8. 部署设计（复用 infra 体系）

### infra/video-to-audio/ 目录结构
```
infra/video-to-audio/
├── docker-compose.yml       # 基础服务定义
├── docker-compose.prod.yml  # 生产端口覆盖
└── docker-compose.dev.yml   # 测试端口覆盖
```

### 关键配置说明
- 前端 Nginx 镜像：托管静态文件 + 反代 `/api/*` 到后端
- 后端需要 `ffmpeg` + `yt-dlp`，基础镜像用 `node:20-alpine` + 额外安装
- 音频文件持久化：`/app/files` 挂载 Docker Volume
- 临时文件目录 `/tmp/vta` 无需持久化，容器内即可

### Nginx 额外配置（相比 security-quiz-game）
```nginx
# 大文件上传支持（分片 5MB × N）
client_max_body_size 20M;

# SSE 长连接：关闭缓冲，确保实时推送
location /api/convert/progress/ {
    proxy_pass http://backend:3000;
    proxy_buffering off;
    proxy_cache off;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    chunked_transfer_encoding on;
}
```

---

## 9. 关键决策记录

| 决策点 | 结论 | 理由 |
|--------|------|------|
| 前端技术栈 | React 19 + Webpack 5 + TypeScript | 用户指定 |
| 视频获取 | yt-dlp（同时支持直链+平台解析） | yt-dlp 天然支持两种场景 |
| 输出格式 | mp3 / aac / wav 用户可选 | 覆盖主流场景 |
| 进度推送 | SSE（Server-Sent Events） | 单向推送，比 WebSocket 简单，够用 |
| 文件上传 | 分片上传（5MB/片，并发3片） | 技术亮点，支持大文件 |
| 用户系统 | JWT 双 Token，硬编码账号，复用 security-quiz-game | 小团队够用，快速上线 |
| 文件存储 | 服务器持久化，每账号最多30条，超限删旧 | 用户可查历史 |
| 数据库 | SQLite（better-sqlite3） | 复用现有体系，无需额外依赖 |

---

## 10. 未来规划（已记录，暂不实现）

1. **批量转换**：多任务并发队列（p-queue 限制并发数），每个任务独立 SSE 进度
2. **时间裁剪 + 多段拼接**：动态表单配置多个输入源（URL/文件）+ 各自时间范围，后端 ffmpeg concat filter 拼接
3. **注册功能**：目前硬编码账号，后续支持用户自助注册
4. **存储扩展**：文件量大后迁移到对象存储（OSS/MinIO）
