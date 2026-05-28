# 音视频转音频工具站 实现任务

## 一、前端项目（video-to-audio）

### 1. 项目脚手架

- [ ] 初始化项目目录 `/Users/meituan/Desktop/video-to-audio/`
- [ ] 创建 `package.json`（React 19、TypeScript、Webpack 5、sass、css-modules）
- [ ] 创建 `tsconfig.json`
- [ ] 创建 `webpack.common.js` / `webpack.dev.js` / `webpack.prod.js`（复用 security-quiz-game 配置）
- [ ] 创建 `babel.config.json`（支持 React 19 JSX transform）
- [ ] 创建 `public/index.html`
- [ ] 创建 `src/index.tsx`（应用入口）
- [ ] 创建 `Dockerfile`（两阶段构建：builder + nginx:alpine，复用 security-quiz-game 模式）
- [ ] 创建 `nginx.conf`（SPA 兜底 + `/api/` 反代 + SSE 无缓冲 + `client_max_body_size 20M`）

### 2. 类型定义

- [ ] 创建 `src/types/convert.ts`（`ConvertState` 状态机类型、`OutputFormat`）
- [ ] 创建 `src/types/history.ts`（`HistoryItem`）
- [ ] 创建 `src/types/sse.ts`（`SSEProgressEvent`、`SSEDoneEvent`、`SSEErrorEvent`）
- [ ] 创建 `src/types/user.ts`（`UserInfo`）

### 3. 工具函数 & Hooks

- [ ] 创建 `src/utils/request.ts`（axios 封装，自动携带 Authorization header，401 时刷新 token，复用 security-quiz-game 模式）
- [ ] 创建 `src/utils/token.ts`（AccessToken 存储/读取/清除，使用 localStorage）
- [ ] 创建 `src/hooks/useSSE.ts`（封装 SSE 连接，自动处理 open/message/error/close）
- [ ] 创建 `src/hooks/useChunkUpload.ts`（分片上传逻辑：文件切片 5MB/片、并发3片上传、进度聚合、自动触发 merge）

### 4. Auth 模块

- [ ] 创建 `src/context/AuthContext.tsx`（全局用户状态 Context：userInfo、accessToken、login、logout）
- [ ] 创建 `src/api/user.ts`（`loginApi`、`logoutApi`、`refreshTokenApi`）
- [ ] 创建 `src/pages/Login/index.tsx`（登录表单：用户名 + 密码，调用 loginApi，成功后跳转 `/`）
- [ ] 创建 `src/pages/Login/index.module.scss`（登录页样式：居中卡片）
- [ ] 创建 `src/components/PrivateRoute.tsx`（路由守卫：未登录跳转 `/login`）

### 5. 主页 - ConvertPanel 区域

- [ ] 创建 `src/api/convert.ts`（`submitUrlConvert`、`initUpload`、`uploadChunk`、`mergeUpload`）
- [ ] 创建 `src/pages/Home/index.tsx`（主页容器，上下两栏布局）
- [ ] 创建 `src/pages/Home/index.module.scss`
- [ ] 创建 `src/pages/Home/components/ConvertPanel/index.tsx`
  - 包含 Tab 切换：URL 模式 / 文件上传模式
  - 使用 `useReducer` 管理 `convertState`（idle → uploading/submitting → converting → done/error）
- [ ] 创建 `src/pages/Home/components/ConvertPanel/UrlInput.tsx`
  - URL 输入框 + 输出格式选择（mp3/aac/wav）+ 提交按钮
  - 提交后调用 `submitUrlConvert`，拿到 `taskId` 后触发 SSE
- [ ] 创建 `src/pages/Home/components/ConvertPanel/FileUpload.tsx`
  - 拖拽/点击上传区域
  - 调用 `useChunkUpload` 执行分片上传
  - 上传完成后调用 `mergeUpload` 拿到 `taskId`，触发 SSE
- [ ] 创建 `src/pages/Home/components/ConvertPanel/ProgressBar.tsx`
  - 通用进度条组件，接收 `percent`（0-100）和 `label`（"上传中..." / "转码中..."）
- [ ] 创建 `src/pages/Home/components/ConvertPanel/ResultPanel.tsx`
  - 显示转换结果：HTML5 `<audio>` 播放器（预览）+ 下载按钮
  - 调用 `/api/file/:fileId/preview` 和 `/api/file/:fileId/download`

### 6. 主页 - HistoryList 区域

- [ ] 创建 `src/api/history.ts`（`getHistory`、`deleteHistory`）
- [ ] 创建 `src/pages/Home/components/HistoryList/index.tsx`
  - 登录后自动加载，转换完成后刷新
  - 空态提示：「暂无转换记录」
- [ ] 创建 `src/pages/Home/components/HistoryList/HistoryItem.tsx`
  - 展示：原始名称、输出格式、时长、文件大小、创建时间
  - 操作：预览（内联播放器展开）、下载、删除（二次确认）
- [ ] 创建 `src/pages/Home/components/HistoryList/index.module.scss`

### 7. 路由配置

- [ ] 创建 `src/router/index.tsx`
  - `/login` → `Login` 页（无守卫）
  - `/` → `Home` 页（`PrivateRoute` 包裹）
  - 其他路径重定向到 `/`

### 8. App 入口

- [ ] 创建 `src/App.tsx`（`AuthContext.Provider` 包裹 `RouterProvider`）

---

## 二、后端项目（video-to-audio-backend）

### 1. 项目脚手架

- [ ] 初始化项目目录 `/Users/meituan/Desktop/video-to-audio-backend/`
- [ ] 创建 `package.json`（express、better-sqlite3、fluent-ffmpeg、uuid、bcryptjs、jsonwebtoken、multer、dotenv）
- [ ] 创建 `tsconfig.json`
- [ ] 创建 `Dockerfile`（两阶段构建，运行阶段额外安装 `ffmpeg` 和 `yt-dlp`）
- [ ] 创建 `src/app.ts`（Express 应用入口，复用 security-quiz-game 结构）

### 2. 数据库

- [ ] 创建 `src/database/index.ts`（SQLite 连接，复用 better-sqlite3）
- [ ] 创建 `src/database/user/index.ts`（`initUserTable`、`seedPresetUsers`，硬编码账号从环境变量读取）
- [ ] 创建 `src/database/task/index.ts`（`initTaskTable`、`createTask`、`updateTask`、`getTask`）
- [ ] 创建 `src/database/history/index.ts`（`initHistoryTable`、`createHistory`、`getUserHistory`、`deleteHistory`、`enforceHistoryLimit`）

### 3. 工具函数

- [ ] 创建 `src/utils/jwt.ts`（复用 security-quiz-game 的 `generateTokens` / `verifyToken`）
- [ ] 创建 `src/utils/response.ts`（复用 `success` / `fail`）
- [ ] 创建 `src/utils/fileId.ts`（生成 fileId：`uuid v4`）
- [ ] 创建 `src/utils/cleanup.ts`（清理临时文件工具函数）

### 4. 中间件

- [ ] 创建 `src/middleware/auth.ts`（复用 security-quiz-game 的 JWT 鉴权中间件）
- [ ] 创建 `src/middleware/errorHandler.ts`（全局错误处理）

### 5. Auth 模块（复用 security-quiz-game）

- [ ] 创建 `src/services/user/index.ts`（`login`、`refreshToken`、`logout`，无注册功能）
- [ ] 创建 `src/controllers/user/index.ts`（`login`、`refresh`、`logout`、`getProfile`）
- [ ] 创建 `src/routes/user/index.ts`

### 6. 转换核心模块

- [ ] 创建 `src/services/convert/ytdlp.ts`（封装 yt-dlp 调用：子进程执行，解析进度输出，支持 progress callback）
- [ ] 创建 `src/services/convert/ffmpeg.ts`（封装 fluent-ffmpeg 调用：输入文件 → 输出格式，支持 progress callback）
- [ ] 创建 `src/services/convert/index.ts`（串联 yt-dlp + ffmpeg，维护 SSE 客户端 Map）
- [ ] 创建 `src/controllers/convert/index.ts`
  - `submitUrl`：接收 URL + format，创建 task，异步启动转换，返回 taskId
  - `initUpload`：初始化分片上传，返回 uploadId
  - `uploadChunk`：接收单片，写入临时目录
  - `mergeAndConvert`：合并分片，创建 task，异步启动转换，返回 taskId
  - `getProgress`：建立 SSE 连接，注册到 SSE 客户端 Map
- [ ] 创建 `src/routes/convert/index.ts`

### 7. 文件服务模块

- [ ] 创建 `src/controllers/file/index.ts`
  - `preview`：流式返回音频，支持 Range 请求
  - `download`：设置 `Content-Disposition: attachment` 后流式返回
- [ ] 创建 `src/routes/file/index.ts`

### 8. 历史记录模块

- [ ] 创建 `src/services/history/index.ts`（`getHistory`、`deleteHistory`，删除时同步删除文件）
- [ ] 创建 `src/controllers/history/index.ts`
- [ ] 创建 `src/routes/history/index.ts`

### 9. 路由注册

- [ ] 更新 `src/routes/index.ts`（注册 `/user`、`/convert`、`/file`、`/history`）

---

## 三、infra 配置（video-to-audio）

- [ ] 创建 `infra/video-to-audio/docker-compose.yml`（基础服务定义：frontend + backend，复用 security-quiz-game 结构）
- [ ] 创建 `infra/video-to-audio/docker-compose.prod.yml`（生产端口：前端 80，挂载 prod volume）
- [ ] 创建 `infra/video-to-audio/docker-compose.dev.yml`（测试端口：前端 8081，挂载 dev volume）

---

## 完成标记说明

所有任务完成后将 `- [ ]` 改为 `- [x]`
