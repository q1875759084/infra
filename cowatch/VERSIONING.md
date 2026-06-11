# CoWatch 版本 Tag 规范

前后端各自独立打 tag，同一里程碑使用相同的 tag 名，便于跨仓库对比。

---

## Tag 命名规则

```
v{major}.{minor}-{slug}
```

| 部分 | 说明 |
|------|------|
| `major` | 大版本，架构级变更时递增 |
| `minor` | 小版本，功能迭代时递增 |
| `slug` | 简短描述，kebab-case，说明该版本的核心内容 |

**示例**

| Tag | 含义 |
|-----|------|
| `v1.0-pre-hls` | HLS 改造前的基线版本（inFlight 方案，mp4 直传） |
| `v1.1-hls` | HLS 切片方案上线 |
| `v1.2-idb-cache` | IDB 分块缓存上线 |
| `v2.0-refactor-xxx` | 大规模重构 |

---

## 打 Tag 流程

```bash
# 前端
cd /Users/meituan/Desktop/CoWatch
git tag v1.x-slug
git push origin v1.x-slug

# 后端
cd /Users/meituan/Desktop/CoWatch-backend
git tag v1.x-slug
git push origin v1.x-slug
```

---

## 版本历史

| Tag | 日期 | 仓库 | 说明 |
|-----|------|------|------|
| `v1.0-pre-hls` | 2026-06-11 | 前端 + 后端 | HLS 改造前基线；SW 采用 inFlight 去重方案，上传支持白名单直传和 proxy 两种模式 |

---

## 对比命令

```bash
# 查看两个版本之间的文件变动列表
git diff v1.0-pre-hls v1.1-hls --stat

# 查看完整 diff
git diff v1.0-pre-hls v1.1-hls

# 查看某个文件在两个版本间的变化
git diff v1.0-pre-hls v1.1-hls -- src/sw.ts
```
