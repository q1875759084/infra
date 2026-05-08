# Incident Report / Bug 追踪记录

> 记录项目业务功能相关的真实问题、根因分析和修复方案。
> CI/CD 基础设施相关问题记录在 `EVOLUTION.md`。

---

## INC-001 · 种子数据未写入：游戏节点数据为空，核心功能不可用

| 字段 | 内容 |
|---|---|
| **事故等级** | P0 |
| **事故类型** | 功能缺失 |
| **发现时机** | 线上首次部署后验证 |
| **影响范围** | 点击任意剧本后页面显示"暂无剧本内容"，游戏无法进行，核心功能完全不可用 |

### 现象
线上请求 `/api/scripts/capture1/nodes/chapter1_node2` 返回节点不存在，游戏页面降级渲染错误提示。

### 排查过程
1. 确认 `seedDatabase()` 已在 `app.ts` 启动时调用 ✅
2. 确认 seed 文件有幂等保护（`INSERT OR IGNORE`）✅
3. 怀疑数据文件未被正确打包 → 查看 Dockerfile，发现 `COPY src/data ./src/data`
4. **关键发现**：seed.ts 编译后路径为 `dist/database/seed.js`，`__dirname` = `/app/dist/database`，`path.resolve(__dirname, '../data/chapter1.json')` 解析为 `/app/dist/data/chapter1.json`，但数据文件被复制到了 `/app/src/data/`，路径不匹配
5. seed 执行时 `fs.existsSync` 检查失败，静默跳过节点插入，无任何报错

### 根因
TypeScript 多阶段构建中，源码路径（`src/`）和编译产物路径（`dist/`）不一致。Dockerfile 中数据文件的 `COPY` 目标路径未与 `__dirname` 的运行时值对齐。

静默跳过的容错逻辑（`console.warn` + `continue`）掩盖了根因，使问题在启动日志中不可见。

### 修复
```dockerfile
# 修复前
COPY src/data ./src/data

# 修复后：对齐编译后 __dirname 的实际路径
COPY src/data ./dist/data
```

### 经验
- 多阶段构建时，**运行时的文件路径以 `dist/` 为基准**，不是 `src/`
- 涉及文件读取时，应在容器内实际验证路径：`docker exec -it 容器名 sh && ls /app/dist/data/`
- 初始化类操作（seed、migration）失败时应 **Fail Fast**（抛错而非静默跳过），方便第一时间发现问题

---

## INC-002 · 未登录用户进入游戏：401 错误未正确降级，页面卡死

| 字段 | 内容 |
|---|---|
| **事故等级** | P1 |
| **事故类型** | 边界场景处理缺失 |
| **发现时机** | 线上首次验证 |
| **影响范围** | 未登录用户点击剧本后页面空白/降级渲染，无法进入游戏 |

### 现象
未登录用户点击剧本 → 触发 `GET /api/records?scriptId=xxx` → 后端返回 401 → 前端 catch 块执行降级 → 但游戏页面仍显示异常状态。

### 根因（待深入确认）
`catch` 块已有降级逻辑（从入口节点开始），理论上应正常工作。初步判断是降级时 `scriptMeta.entryNodeId` 与节点 API 时序问题，或节点 API 本身返回异常。需结合 INC-001 数据库问题排查，INC-001 修复后重新验证此问题是否复现。

### 现有代码（已有兜底）
```typescript
try {
  const res = await getRecord(scriptId);
  // ...
} catch {
  // 读档失败降级：从入口节点开始，不影响游戏
  if (scriptMeta.entryNodeId) {
    loadNode(scriptMeta.entryNodeId);
  }
}
```

### 后续
INC-001 修复部署后重新验证，确认是独立问题还是数据库为空的连锁反应。

---

## 事故统计

| 等级 | 数量 | 说明 |
|---|---|---|
| P0 | 1 | 核心功能不可用（INC-001） |
| P1 | 1 | 主要功能受损（INC-002，待确认） |

## 待跟进

- [ ] INC-001 修复部署后，验证 INC-002 是否为独立 bug
- [ ] 确认未登录用户完整游玩流程是否正常（无存档场景）
