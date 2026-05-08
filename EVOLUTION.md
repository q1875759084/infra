# CI/CD 架构演化过程

## 阶段一：最初状态（Docker 单容器）

### 架构
```
服务器
  └── 前端容器（Nginx + 静态文件）
        └── proxy_pass → 172.17.0.1:3001（宿主机上手动跑的 Node.js）
```

### 问题
- 后端跑在宿主机，不在容器里，无法随代码自动部署
- `172.17.0.1` 是 Docker 网桥 IP，后端容器化后这个地址失效
- 前端 CI/CD 用 `docker run`，每次部署先 `docker stop` + `docker rm` + `docker run`

### 前端 deploy.yml 的部署步骤
```
构建镜像 → 推 Docker Hub → SSH 到服务器：
  docker stop security-quiz-game
  docker rm security-quiz-game
  docker run -d -p 80:80 ...
```

---

## 阶段二：引入 Docker Compose（多容器编排）

### 为什么需要 Compose
- 后端也要容器化，前后端两个容器需要互相通信
- 两个容器不能用 IP 通信（IP 会变），需要用**服务名**作为 DNS
- Compose 创建私有网络，容器间用服务名互相访问：`proxy_pass http://backend:3000`

### 架构
```
Compose 私有网络
  ├── frontend 容器（Nginx，ports: 80:80 对外暴露）
  │     └── proxy_pass http://backend:3000
  └── backend 容器（Node.js，expose: 3000 只在内部可访问）
```

### Compose 多环境方案
一个基础文件 + 环境覆盖文件，合并执行：
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d  # 生产
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d   # 测试
```

| 文件 | 内容 |
|---|---|
| `docker-compose.yml` | 服务定义、网络（环境无关） |
| `docker-compose.prod.yml` | 差异：`ports: 80:80` + backend volume: `quiz-db-data-prod` |
| `docker-compose.dev.yml` | 差异：`ports: 8080:80` + backend volume: `quiz-db-data-dev` |

**环境数据隔离**：volume 名称在覆盖文件中各自声明，生产和测试数据库完全独立。若放在基础文件共用同一个 volume，测试环境的数据操作会直接污染生产数据库。

### compose 文件放在哪里
**错误做法**：放在前端业务仓库 → 业务仓库承担了部署职责，耦合

**正确做法**：放在独立的 infra 仓库 → 部署配置和业务代码分离

### 前端 deploy.yml 的部署步骤
```
构建镜像 → 推 Docker Hub
  → scp compose 文件到服务器
  → SSH：docker compose up -d frontend
```

---

## 阶段三：引入 infra 仓库（GitOps 雏形）

### 为什么需要 infra 仓库
- 有多个项目时，每个项目都有 compose 文件，分散在各业务仓库难以管理
- Secrets（服务器地址、JWT_SECRET 等）在每个仓库里都要配一遍，维护成本高
- 部署逻辑和业务逻辑应该分离：业务仓库只管"构建"，infra 仓库只管"部署"

### 最终架构
```
security-quiz-game（前端业务仓库）
  职责：CI 校验 + 构建镜像 + 推 Docker Hub + 触发 infra
  Secrets：DOCKER_HUB_USERNAME / DOCKER_HUB_TOKEN / INFRA_DEPLOY_TOKEN

security-quiz-game-backend（后端业务仓库）
  职责：CI 校验 + 构建镜像 + 推 Docker Hub + 触发 infra
  Secrets：DOCKER_HUB_USERNAME / DOCKER_HUB_TOKEN / INFRA_DEPLOY_TOKEN

infra（部署仓库）
  职责：接收通知 + 传 compose 文件到服务器 + 执行 docker compose up -d
  Secrets：SERVER_HOST / SERVER_USER / SERVER_SSH_KEY（服务器相关）
           DOCKER_HUB_USERNAME（拉镜像用）
           CORS_ORIGIN / JWT_SECRET（注入给容器的环境变量）
```

### 跨仓库触发机制（repository_dispatch）
业务仓库 push tag 后，最后一步通知 infra 仓库：
```yaml
- uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.INFRA_DEPLOY_TOKEN }}  # 有 infra 仓库写权限的 PAT
    repository: yourname/infra
    event-type: deploy-frontend               # infra 监听这个事件类型
    client-payload: |
      {
        "image_tag": "v1.0.0",
        "environment": "prod"
      }
```

infra 仓库监听事件，拿到 image_tag 和 environment，执行对应的 compose 部署。

### 部署流程（最终版）

#### 测试环境发布（workflow_dispatch 手动触发）
```
1. 开发者在 GitHub Actions 页面手动触发 workflow_dispatch，选择目标分支

2. 前端仓库 deploy-dev job：
   → 先跑 CI 校验（lint + build），机制保障，不靠人记
   → 构建镜像，只推一个 tag：sha（不可变，精确对应这次 commit）
     yourname/security-quiz-game:<sha>
   → repository_dispatch → infra 仓库
     payload: { image_tag: "<sha>", environment: "dev" }

3. infra 仓库 deploy-frontend.yml 触发：
   → checkout infra 代码（含 compose 文件）
   → scp：把 docker-compose.yml + docker-compose.dev.yml 传到服务器 ~/deploy/
   → SSH：服务器执行
     FRONTEND_TAG=<sha> docker compose \
       -f ~/deploy/security-quiz-game/docker-compose.yml \
       -f ~/deploy/security-quiz-game/docker-compose.dev.yml \
       up -d frontend
     （只重启 frontend 容器，backend 不受影响）
     （测试环境端口 8080）
```

#### 生产环境发布（push tag v*.*.*）
```
前提：测试环境已用 sha tag 验证通过

1. 开发者：git tag v1.0.0 && git push origin v1.0.0

2. 前端仓库触发两个 job：
   → ci job：lint + build 校验（needs: ci 强制依赖，tag 发布也要校验）
   → deploy-prod job（needs: ci，ci 通过才执行）：
     不重新构建镜像（核心：测试的和上线的必须是同一个产物）
     给已有的 sha 镜像追加语义版本 tag（纯 manifest 操作）：
       docker buildx imagetools create \
         --tag yourname/security-quiz-game:v1.0.0 \
         yourname/security-quiz-game:<sha>
     → repository_dispatch → infra 仓库
       payload: { image_tag: "v1.0.0", environment: "prod" }

3. infra 仓库 deploy-frontend.yml 触发：
   → checkout infra 代码
   → scp：把 docker-compose.yml + docker-compose.prod.yml 传到服务器 ~/deploy/
   → SSH：服务器执行
     FRONTEND_TAG=v1.0.0 docker compose \
       -f ~/deploy/security-quiz-game/docker-compose.yml \
       -f ~/deploy/security-quiz-game/docker-compose.prod.yml \
       up -d frontend
     （生产环境端口 80）
```

#### 一次构建原则的意义
```
❌ 错误做法（之前的版本）：
  测试：构建一次，推 sha tag
  生产：重新构建一次，推 v1.0.0 tag
  → 两次构建产物不保证完全一致（构建环境、依赖版本可能漂移）
  → 测试通过的不是上线的那个东西

✅ 正确做法（现在的版本）：
  测试：构建一次，推 sha tag，部署验证
  生产：不构建，给同一个 sha 镜像追加 v1.0.0 tag，部署
  → 测试和生产使用完全相同的二进制产物
  → 只有环境变量（CORS_ORIGIN、数据库地址等）不同
```

### 独立发布的技术基础
`docker compose up -d frontend` 或 `up -d backend`：
- 只重启指定服务的容器
- 其他服务继续运行，用户无感知
- 这就是"前后端独立发布"的实现原理

---

## 阶段四：CD 触发机制重构（Promotion-based Deployment）

### 问题：push tag 自动触发 CD 的隐患

阶段三的生产部署由 `push tags: v*.*.*` 触发，存在根本性设计缺陷：

```
❌ push tag 自动触发 CD 的问题：
  - tag 一旦 push 到远程，无法撤回
  - CI/Actions 立即执行，没有人工卡点
  - "版本归档"（打 tag）和"部署决策"（执行 CD）被耦合到同一个 Git 操作
  - 误操作 push tag → 生产事故，且无法中止
```

### 认知纠正：tag 是什么？

tag **不是对分支的说明**，而是**对某一个具体 commit 的永久标记**：

```
commit A → commit B → commit C  ← main（会移动）
                           ↑
                        v1.0.0   ← tag（永远不动，版本归档）
```

- **分支指针**：随新 commit 自动前移，表示"当前最新状态"
- **tag 指针**：永远指向打 tag 时的那个 commit，表示"这个快照是 v1.0.0"

tag 的语义是**版本归档**，不是部署指令。让 push tag 自动触发生产部署，等于把"盖章"和"执行"强制绑定，违背职责分离原则。

### 解决方案：Promotion-based Deployment（晋级式部署）

**核心思想**：CI 和 CD 是两个独立的关注点，触发机制完全分离。

```
push 分支     → 触发 CI（只读操作：lint + build）
workflow_dispatch → 触发 CD（写操作：改变服务器状态）
git push tag  → 什么都不触发（零副作用，只是 Git 历史归档）
```

| 事件 | 触发 job | 有无副作用 | 说明 |
|------|---------|---------|------|
| `push 分支` | CI | 无 | 只读，检查代码质量 |
| `workflow_dispatch(deploy-dev)` | CD-测试 | 有 | 构建镜像、改变测试服务器状态 |
| `workflow_dispatch(deploy-prod)` | CD-生产 | 有 | 改变生产服务器状态 |
| `git push tag` | 无 | 无 | 版本归档，零副作用 |

### 新的 workflow_dispatch 输入参数设计

```yaml
on:
  push:
    branches: ['**']        # 只监听分支 push，触发 CI
  workflow_dispatch:
    inputs:
      event:
        description: '选择操作类型'
        type: choice
        options: [deploy-dev, deploy-prod]
      tag:
        description: '生产部署时填写，对应测试验证过的 tag，如 v1.0.0'
        required: false
```

### 新的完整操作流程

```
1. push 代码 → CI 自动运行（lint + build），代码质量卡点

2. 准备给 QA 测试
   → GitHub Actions → Run workflow → event=deploy-dev
   → 构建镜像 sha-abc123，部署测试环境

3. QA 测试通过，准备发布
   → git tag -a v1.0.0 -m 'release: xxx'
   → git push origin v1.0.0
   → （什么都不触发，只是版本归档）

4. 上线决策（人工触发）
   → GitHub Actions → Run workflow
   → event=deploy-prod，tag=v1.0.0
   → 给 sha-abc123 追加 v1.0.0 tag（imagetools create）
   → 部署生产
```

### deploy-prod 如何找到 tag 对应的 sha？

```yaml
- name: Checkout 代码（用于获取 tag 对应的 sha）
  uses: actions/checkout@v4
  with:
    ref: ${{ inputs.tag }}   # 检出指定 tag，github.sha 就是该 tag 对应的 commit hash
```

checkout 指定 tag 后，`${{ github.sha }}` 就是该 tag 打在的那个 commit 的 hash，与测试阶段推送的 sha 镜像完全对应，一次构建原则得以保证。

### 防御性校验：tag 参数不能为空

```yaml
- name: 校验 tag 参数不能为空
  run: |
    if [ -z "${{ inputs.tag }}" ]; then
      echo "❌ 生产部署必须填写 tag 参数，如 v1.0.0"
      exit 1
    fi
```

`workflow_dispatch` 的 `choice` 类型字段不能与 `required` 混用，必须手动校验，让错误提前暴露，而非在 `imagetools create` 报出难以理解的错误。

### 与大厂实践的对应

大厂完整的上线审批链路（参考）：

```
代码合并 main
  → CI 自动运行
  → 开发者发起"上线申请"（填写变更范围、风险评估）
  → 技术负责人审批
  → 审批通过，系统自动或人工触发 CD
  → 灰度发布（1% → 10% → 100% 流量）
  → 监控观察，无异常后全量
```

当前项目实现了其中的核心机制：**CI 与 CD 完全解耦，CD 必须人工触发**。
`environment: production` + `required reviewers`（GitHub 原生审批功能）可在此基础上直接叠加，无需改造触发机制。

---

## 关键概念速查

| 概念 | 说明 |
|---|---|
| `ports` vs `expose` | ports 对外暴露，expose 只在 Compose 内部网络可见 |
| `docker compose up -d [service]` | 只重启指定服务，实现独立发布 |
| `repository_dispatch` | GitHub 跨仓库触发 Actions 的机制，本质是一个 HTTP POST 请求 |
| `INFRA_DEPLOY_TOKEN` | 有 infra 仓库 Actions write 权限的 PAT，用于跨仓库触发 |
| `scp` | Secure Copy Protocol，基于 SSH 的文件传输协议 |
| named volume | Docker 管理的持久化存储，与容器生命周期解绑，`docker rm` 不会删除 |
| 环境变量三层传递 | GitHub Secret → runner → compose `${}` → 容器 `process.env` |
| `docker buildx imagetools create` | 给已有镜像追加新 tag，纯 manifest 操作，不重新构建，实现一次构建原则 |
| sha tag | 以 commit hash 命名的镜像 tag，不可变，精确对应某次代码提交，用于测试环境 |
| 语义版本 tag（v1.0.0） | 人工打 tag 时追加，用于生产环境，表示"经过测试确认可发布的版本" |
| 禁用 latest tag | latest 语义模糊，多环境下无法分辨指向哪个版本，大厂规范禁用 |
| ci job 限定 `push` 事件 | `if: github.event_name == 'push'`，避免 workflow_dispatch 触发时跑两遍 CI |
| Promotion-based Deployment | 晋级式部署：代码经过测试环境验证后，人工决策"晋级"到生产环境，CI 与 CD 触发机制完全分离 |
| push tag 零副作用 | tag 是版本归档，不是部署指令；push tag 不触发任何 Actions，部署必须单独人工触发 |
| `workflow_dispatch inputs` | GitHub Actions 手动触发时的参数输入，支持 `choice`（下拉选择）和 `string`（文本输入）类型 |
| `actions/checkout ref` | `ref: ${{ inputs.tag }}` 检出指定 tag，使 `github.sha` 对应该 tag 的 commit，用于 imagetools create 找到正确的 sha 镜像 |
| 有无副作用的区分 | CI 是只读操作（读代码、跑校验），CD 是写操作（改变基础设施状态）；两者触发机制应分离 |

---

## 阶段五：Workflow 重复代码消除

### 问题：CI steps 在多个 job 中重复

改造前，`ci` 和 `deploy-dev` 两个 job 各自包含完全相同的 5 个 steps（checkout、node 安装、依赖安装、lint、build），每个仓库重复一次，两个仓库共重复 4 次。

### 三种解法的对比

#### 方案 X：YAML Anchors（不可用）
YAML 语法原生支持锚点 `&` 和引用 `*` 复用片段，但 **GitHub Actions 不支持**，会直接报错拒绝解析。

#### 方案 A：ci job 的 if 条件合并两种事件（采用）
```yaml
ci:
  if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'

deploy-dev:
  needs: ci   # 等 ci 完成再执行，无需内联 CI steps
  if: ... && inputs.event == 'deploy-dev'

deploy-prod:
  needs: ci   # 同样约束，防止绕过 CI 直接部署生产
  if: ... && inputs.event == 'deploy-prod'
```
- 单文件，无需新增文件
- `deploy-dev` / `deploy-prod` 只写独有步骤，重复消除
- **注意**：`deploy-prod` 必须同样加 `needs: ci`，否则生产部署绕过 CI 校验

#### 方案 B：同仓库 Reusable Workflow
把 CI steps 抽成独立的 `_ci.yml`，用 `workflow_call` 事件声明为可复用，其他 job 通过 `uses: ./.github/workflows/_ci.yml` 调用。
- 结构最清晰，但多一个文件，跳转阅读
- 适合 2-3 个仓库有大量共同逻辑时

#### 方案 C：跨仓库公共 Workflow 仓库（大厂规范）
把通用 workflow 放到独立的 `org/shared-workflows` 仓库，业务仓库通过 `uses: org/shared-workflows/.github/workflows/node-ci.yml@v2.1.0` 跨仓库调用，传入差异化参数（node 版本、lint 命令等）。
- **本质是"样板 yml + 项目差异化配置"**
- 公共 workflow 需要版本管理和向后兼容，本身是一个内部 SDK
- 适合 5+ 个仓库，ROI 才合理

### 演进路径

```
1 个项目      → 方案 A（ci if 合并，单文件，够用）
2-3 个项目    → 方案 B（同仓库 Reusable Workflow）
5+ 个项目     → 方案 C（公共仓库 shared-workflows）
大厂（几百个）→ 自研 CI 平台（Jenkins/Tekton），workflow 只是入口
```

架构决策原则：不是一上来就做最重的方案，而是清楚当前方案的边界在哪，什么时候该演进。

### 当前采用方案 A 的结果

```
改造前：ci job + deploy-dev job 各有 5 个重复 steps（约 20 行 × 2）
改造后：ci job 统一处理所有事件的校验，deploy-dev/prod 只写独有逻辑
减少约 30 行，且无新增文件
```

---

## 阶段六：端口契约收敛

### 问题：端口号 3000 散落在三处

```
nginx.conf         → proxy_pass http://backend:3000
docker-compose.yml → expose: "3000" 和 PORT=3000
后端 app.ts        → app.listen(process.env.PORT || 3000)
```

修改端口时需要同步改三处，存在遗漏风险。

### 解决方案：收敛到 compose 环境变量

```yaml
# docker-compose.yml
backend:
  expose:
    - "${BACKEND_PORT:-3000}"      # 默认值 3000，可通过环境变量覆盖
  environment:
    - PORT=${BACKEND_PORT:-3000}   # 注入给容器，后端读 process.env.PORT
```

后端 `app.ts` 读取 `process.env.PORT`，已与具体数值解耦。

### nginx.conf 的局限性

`nginx.conf` 是静态文件，**无法读取 compose 环境变量**，`proxy_pass http://backend:3000` 中的端口只能硬编码。

解决方式：在 `nginx.conf` 中添加注释，明确声明此处端口必须与 `BACKEND_PORT` 默认值保持一致，修改时两处同步。

```nginx
# 端口 3000 与 docker-compose.yml 中 BACKEND_PORT 默认值保持一致
# nginx.conf 是静态文件，无法读取 compose 环境变量，因此此处硬编码
# 若需修改端口，必须同步修改：nginx.conf（此处）+ compose BACKEND_PORT 默认值
proxy_pass http://backend:3000;
```

### 前后端端口耦合的本质

`nginx → backend:3000` 是**边界契约**（interface contract），不可消除。前端容器必须知道后端的地址和端口才能转发请求，这是协作的必要条件。

真正需要管理的是：**契约的定义有单一来源**。当前收敛后：
- 端口值定义在 `BACKEND_PORT` 环境变量（单一来源）
- `nginx.conf` 的硬编码是已知的例外，有注释说明
- 规模更大时可用服务发现（Consul）或配置中心彻底消除

---

## 阶段七：环境变量管理规范

### 环境变量的判断标准（12-Factor App）

满足以下任一条件的值，必须外置为环境变量，不得硬编码：
1. **不同环境值不同**（测试 CORS 地址 vs 生产 CORS 地址）
2. **包含凭证/密钥**（绝对不能进代码仓库）
3. **运维需要修改但不想触发重新发布**

> 快速判断：如果这个值出现在 GitHub 上会让你感到不安，它就应该是 Secret。

### 构建时变量 vs 运行时变量

这是前后端最根本的差异，不是"有没有 return"，而是**消费变量的进程和 compose 注入的容器是否同一个**：

```
前端（webpack 构建时消费）：
  GitHub Actions 注入 → webpack 进程读取 → DefinePlugin 烧录进 dist/xxx.js → webpack 退出
  之后 nginx 容器启动 → compose 注入环境变量 → nginx 完全不认识 → 被忽略
  dist/ 里的值永远是构建时烧录的那个，运行时无法更改

后端（express 运行时消费）：
  compose 注入 → 容器启动 → express 进程读取 process.env → 实时生效
  改了环境变量重启容器即可，不需要重新构建镜像
```

| | 前端 | 后端 |
|---|---|---|
| 变量消费时机 | 构建时（webpack 运行时） | 运行时（express 运行时） |
| 变量来源 | CI runner 环境 / `.env.development` | compose `environment` / `.env` |
| 改变量是否需要重新构建 | **是**，值已烧录进 JS 文件 | **否**，重启容器即可 |
| 变量是否对外暴露 | **是**，打包进 JS 任何人可见 | **否**，只在服务器进程内 |

**结论**：前端变量绝对不能放密钥；后端密钥通过 compose 注入，不进镜像不进代码。

### 本地开发环境变量加载

**前端（webpack.common.js）**：
```javascript
// 用 DEPLOY_ENV 区分 CI 和本地，比 NODE_ENV 更准确
// NODE_ENV 只反映 webpack 构建模式，无法区分业务环境
// DEPLOY_ENV 只有 CI 平台才会注入，本地不存在
const isCI = !!process.env.DEPLOY_ENV;
if (!isCI) {
  require('dotenv').config({ path: '.env.development' });
}
```

**后端（app.ts 第一行）**：
```typescript
import 'dotenv/config';
// 有 .env 文件就读取，没有就静默跳过（无副作用）
// 容器里 .env 已被 .dockerignore 排除，此行在容器中等同于 no-op
```

两种方式都正确，适合各自场景：
- 前端用 `DEPLOY_ENV` 显式判断，因为构建工具需要精确区分"CI 构建"和"本地构建"
- 后端用文件存在性隐式判断，因为容器里不存在 `.env` 文件本身就是天然开关

### Fail Fast 原则

配置缺失时的处理方式取决于"缺失后服务是否仍能正常运行"：

| 变量 | 缺失时行为 | 原因 |
|---|---|---|
| `JWT_SECRET` | 拒绝启动 | 没有它认证功能完全瘫痪，属于核心依赖 |
| `CORS_ORIGIN` | 空列表，跨域全拒 | 同域部署（nginx 反代）时浏览器不触发 CORS，服务仍正常 |
| `PORT` | 回退 `3000` | 任何环境下 3000 都是安全的默认值 |

```typescript
// JWT_SECRET：核心依赖，缺失立即拒绝启动
const SECRET = process.env.JWT_SECRET;
if (!SECRET) {
  throw new Error('JWT_SECRET 环境变量未配置，应用拒绝启动');
}

// CORS_ORIGIN：可选配置，缺失时安全默认（空列表 = 跨域全拒，比允许所有来源更安全）
// 同域部署（nginx 反代 /api/*）时浏览器不触发 CORS，空列表不影响功能
// 支持多域名白名单，逗号分隔：CORS_ORIGIN=https://a.com,https://b.com
const corsOrigins = process.env.CORS_ORIGIN
  ? process.env.CORS_ORIGIN.split(',').map(o => o.trim())
  : [];

// PORT：有安全兜底值
const port = Number(process.env.PORT) || 3000;
```

**判断标准**：不是"不同环境值不同就要 Fail Fast"，而是"缺失后服务核心功能是否瘫痪"。

### CORS 白名单的管理策略

`CORS_ORIGIN` 是业务配置而非安全凭证（泄露无安全风险），但放在 Secrets 的理由是：
- 不同环境值不同（本地 localhost，生产真实域名）
- 增加白名单时只需修改 Secrets 并重新触发部署，无需改代码或重新构建镜像
- 运维操作，不触碰任何代码仓库

本地 `.env` 开发者自行维护需要的 localhost 端口，不依赖后端代码特判：
```
CORS_ORIGIN=http://localhost:5173,http://localhost:3001
```

生产 Secrets 只填真实域名：
```
CORS_ORIGIN=https://yourdomain.com,https://admin.yourdomain.com
```

**后端代码不包含任何环境判断逻辑**（如 `if localhost` 特判），环境差异完全由配置值决定，这是关注点分离的体现。

企业级演进路径：Secrets → GitHub Variables（非敏感配置语义更清晰）→ 配置中心（热更新，零停机，有审批记录）。

### 白名单三种存放位置的对比

| 方案 | 增加白名单需要 | 适用场景 |
|---|---|---|
| **硬编码在后端代码** | 改代码 → 重新构建镜像 → 发版 | ❌ 不可用，业务配置与代码耦合 |
| **写在 compose yml** | 改 yml → 提交 infra 仓库 → `docker compose up` | 改动有 git 记录，适合改动频率极低的场景 |
| **写在平台 Secrets/Variables** | 改平台配置 → 手动触发 deploy workflow → 容器重启 | 运维操作，不触碰代码仓库，适合需要独立操作的场景 |
| **配置中心（Apollo/Nacos）** | 配置中心后台修改 → 自动推送 → 热更新 | 零停机，有审批记录，适合企业级多团队协作 |

**当前项目选择 Secrets 的理由**：
- 不需要改任何代码或 yml，运维操作闭环
- 触发重新部署的成本可接受（手动 workflow_dispatch）
- 相比 compose yml，配置变更不与基础设施变更混在同一次 commit 里

**判断"谁来改、多久改一次"**：
- 只有开发者改、极少改 → compose yml（进 git，变更有记录）
- 运维或业务方需要改、较频繁 → Secrets / Variables
- 需要热更新、有审批流 → 配置中心

### .env 文件规范

```
.env            # 本地实际使用，加入 .gitignore 和 .dockerignore，不进仓库不进镜像
.env.example    # 变量名模板，加注释说明用途，提交到仓库，供新开发者参考
```

### 本项目 Secrets 完整清单

**业务仓库（前端 / 后端各自配置）**：

| Secret | 用途 |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub 登录用户名 |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token，推送镜像 |
| `INFRA_REPO_TOKEN` | GitHub PAT，触发 infra 仓库 repository_dispatch |

**infra 仓库**：

| Secret | 用途 |
|---|---|
| `DOCKERHUB_USERNAME` | 拉取私有镜像时登录 |
| `DOCKERHUB_TOKEN` | 拉取私有镜像 |
| `SERVER_HOST` | 服务器 IP |
| `SERVER_USERNAME` | SSH 登录用户名 |
| `SERVER_SSH_KEY` | SSH 私钥（完整内容） |
| `CORS_ORIGIN` | 运行时注入给后端容器，不同环境值不同 |
| `JWT_SECRET` | JWT 签名密钥，必须是强随机字符串，绝不能硬编码 |

> `INFRA_REPO_TOKEN` 是 GitHub PAT（GitHub 颁发），`DOCKERHUB_TOKEN` 是 Docker Hub Access Token（Docker Hub 颁发），两者都叫 Token 但性质完全不同，不可混用。

---

## 阶段八：CI/CD 安全加固

### P0：Secret 不能明文出现在命令行

**问题**：`appleboy/ssh-action` 的 `script` 字段里直接拼接 Secret：

```yaml
# 危险写法
script: |
  JWT_SECRET=${{ secrets.JWT_SECRET }} docker compose up -d
```

`${{ secrets.XXX }}` 在 `script` 字段里会被展开成明文字符串，**整条命令会出现在服务器的进程列表（`ps aux`）和 shell history 里**，任何能登录服务器的人都能看到密钥值。

**修复**：通过 `env` + `envs` 机制，Secret 经加密通道传入 SSH session：

```yaml
env:
  JWT_SECRET: ${{ secrets.JWT_SECRET }}  # runner 层：Secret → runner 环境变量
  CORS_ORIGIN: ${{ secrets.CORS_ORIGIN }}
with:
  envs: JWT_SECRET,CORS_ORIGIN           # ssh-action 加密传入远端 SSH session
  script: |
    docker compose up -d                 # 命令行里只有变量名，值不可见
```

**传递链路**：
```
GitHub Secrets（加密存储）
  → runner 环境变量（内存，不写磁盘）
  → SSH 加密通道
  → 远端 shell 环境变量（内存，不出现在命令行）
  → docker compose 读取
```

### P1：职责分离——前端部署不传后端变量

前端部署 workflow 原来传了 `CORS_ORIGIN` 和 `JWT_SECRET`，但这两个变量只有后端容器消费。`docker compose up -d frontend` 只重启前端容器，这两个变量传了没有任何作用。

修复后：`deploy-frontend.yml` 只传 `DOCKER_HUB_USERNAME`、`FRONTEND_TAG`、`ENVIRONMENT`，注释说明原因。

### P2：action 版本锁定到 commit hash（供应链安全）

`uses: some-action@v3` 中的 `v3` tag 可以被 action 作者随时覆盖指向新 commit，workflow 会在不知情的情况下执行新代码。若 action 仓库被攻击，攻击者可推送恶意代码并覆盖 tag，在 CI 执行时窃取 Secrets。这是**供应链攻击**（Supply Chain Attack）。

```yaml
# 改前（tag 可变，存在供应链风险）
uses: appleboy/ssh-action@v1.0.3

# 改后（hash 不可变，与特定 commit 永久绑定）
uses: appleboy/ssh-action@029f5b4aeeeb58fdfe1410a5d17f967dacf36262  # v1.0.3
```

**P7 架构师能力边界**：
- 必须掌握：P0，理解 Secret 传递机制，能在 CR 时识别明文泄露风险
- 必须了解：P2，知道 tag 可变、hash 不可变、供应链攻击的概念，能在技术评审时识别风险
- 不需要操作：P2 的日常维护（查 hash、更新版本），这是 DevSecOps / 平台团队或 Dependabot 等工具的职责

---

## 附录：三个仓库 Secrets 配置清单

### 前端业务仓库（security-quiz-game）

| Secret 名 | 来源 | 用途 |
|---|---|---|
| `DOCKER_HUB_USERNAME` | Docker Hub 账号名 | `docker login` 登录凭证，`docker tag` 时拼接镜像名 |
| `DOCKER_HUB_TOKEN` | Docker Hub → Account Settings → Security → Access Tokens | 推送镜像到 Docker Hub（比密码更安全，可单独撤销） |
| `INFRA_DEPLOY_TOKEN` | GitHub → Settings → Developer settings → Fine-grained PAT，授权 infra 仓库 Contents: Read and Write | 触发 infra 仓库的 `repository_dispatch` 事件，通知 infra 执行部署 |

### 后端业务仓库（security-quiz-game-backend）

与前端仓库完全相同，三个 Secret 各自独立配置（两个仓库共用同一套值即可）：

| Secret 名 | 来源 | 用途 |
|---|---|---|
| `DOCKER_HUB_USERNAME` | 同上 | 同上 |
| `DOCKER_HUB_TOKEN` | 同上 | 同上 |
| `INFRA_DEPLOY_TOKEN` | 同上（同一个 PAT 可复用） | 同上 |

### infra 仓库

| Secret 名 | 来源 | 用途 |
|---|---|---|
| `DOCKER_HUB_USERNAME` | Docker Hub 账号名 | 若镜像为私有，`docker pull` 前需要登录 |
| `DOCKER_HUB_TOKEN` | Docker Hub Access Token | 拉取私有镜像（公开镜像可省略） |
| `SERVER_HOST` | 云服务器公网 IP | SSH / SCP 连接目标地址 |
| `SERVER_USER` | 服务器登录用户名（如 `ubuntu`、`root`） | SSH 登录身份 |
| `SERVER_SSH_KEY` | 服务器 SSH 私钥完整内容（`~/.ssh/id_rsa` 的内容） | SSH 身份认证，替代密码登录 |
| `CORS_ORIGIN` | 前端域名（如 `https://yourdomain.com`） | 运行时注入给后端容器，控制 CORS 允许的来源；不同环境值不同，不能硬编码 |
| `JWT_SECRET` | 强随机字符串（建议 32 位以上） | JWT 签名密钥；泄露后攻击者可伪造任意用户身份，必须保密 |

### 关键说明

**`INFRA_DEPLOY_TOKEN` 的权限配置**：
- 类型：Fine-grained Personal Access Token（不用 Classic token）
- 授权仓库：只选 `infra`，不授权其他仓库（最小权限原则）
- 权限：`Contents: Read and Write`（`repository_dispatch` API 需要此权限）
- `Metadata: Read-only` 为 GitHub 自动附加，无法去除，无安全风险

**两种 Token 的本质区别**：
- `DOCKER_HUB_TOKEN`：Docker Hub 颁发，控制镜像仓库的读写权限
- `INFRA_DEPLOY_TOKEN`：GitHub 颁发，控制 GitHub 仓库的 API 操作权限
- 两者都叫 Token，但颁发方和控制范围完全不同，不可混用

**Secret 的变量名必须与 yml 文件里的 `secrets.XXX` 完全一致**，大小写敏感。

---

## 待完成

- [ ] 联调验证整条链路

---

## 备案：迁移到 GitHub Environments 管理 Secrets

### 现状

所有 Secrets 平铺在仓库级（Repository Secrets），deploy-dev 和 deploy-prod 两个 job 读取同一套 Secrets，没有环境隔离。

### 方案

创建两个 Environment：`dev` 和 `prod`，按环境分别配置差异化 Secrets：

```
dev  environment:
  CORS_ORIGIN=https://test.yourdomain.com
  JWT_SECRET=test_secret
  （SERVER_HOST 等服务器凭证与 prod 相同时可保留在 Repository Secrets）

prod environment:
  CORS_ORIGIN=https://yourdomain.com
  JWT_SECRET=prod_strong_secret
  Required reviewers: 指定审批人（生产部署必须人工审批才能执行）
```

yml 里每个 job 声明使用的 Environment：

```yaml
deploy-dev:
  environment: dev   # 只能读 dev environment 的 Secrets

deploy-prod:
  environment: prod  # 只能读 prod environment 的 Secrets，且需审批通过
```

### 价值

| 能力 | 说明 |
|---|---|
| 环境隔离 | deploy-dev job 物理上无法读取生产凭证 |
| 审批门禁 | prod environment 配置 Required reviewers，CD 触发后须人工审批 |
| 最小权限 | 每个 job 只能访问自己环境的 Secrets |

### 面试说法

> "生产和测试的凭证通过 GitHub Environments 完全隔离，测试 job 物理上拿不到生产 SSH Key。生产 Environment 配置了 Required reviewers，workflow_dispatch 触发后必须指定人审批才能执行，实现了 CI/CD 的双重人工卡点。"

### 迁移成本

低。每个 deploy job 加一行 `environment: xxx`，然后把差异化 Secrets 从 Repository 级移入对应 Environment 即可，yml 其余逻辑不变。

---

## 运维操作手册：如何查找 SHA tag 并回滚

### 背景

测试环境部署不打语义化版本 tag，只有 `sha-xxxxxxxx` 格式的镜像 tag。  
代码仓库通常只有开发人员有权限，但**重新部署/回滚**的操作者可能是运维、负责人，因此需要能在不访问代码仓库的地方查看历史版本。

### SHA tag 的三个查看入口

| 入口 | 路径 | 适合谁 |
|---|---|---|
| **Docker Hub Tags 页面** | hub.docker.com → 你的仓库 → Tags | 运维/负责人（无需代码权限） |
| **GitHub Actions 运行记录** | 业务仓库 → Actions → 对应的 workflow run → deploy-dev job → `Build and push` 步骤日志 | 开发人员 |
| **Git commit hash** | `git log --oneline -10`，前8位对应 `sha-xxxxxxxx` | 开发人员本地 |

### 在 GitHub Actions 日志里找 SHA

从截图可以看到 `ci` job 只有 lint/build 步骤，没有 Build and push——**镜像构建在 `deploy-dev` job 里**，不在 `ci` job 里。

正确路径：
1. 点左侧 **deploy-dev** job
2. 展开 **Build and push** 步骤
3. 日志里找 `Tags:` 行，格式为：
   ```
   Tags: yourname/security-quiz-game:sha-a1b2c3d4
   ```
   `sha-a1b2c3d4` 就是填入 infra workflow 的值。

### 可追溯性（Traceability）

Git commit → CI 构建 → Docker tag 三者是同一版本的不同视角：

```
git commit: a1b2c3d4
     ↓ CI 构建
Docker tag: sha-a1b2c3d4
     ↓ infra 部署
服务器运行的容器
```

出了问题可以从任意入口追溯到另外两个，这是可追溯性设计的体现。

### infra 仓库手动部署的三种场景

| 场景 | image_tag 填什么 | 说明 |
|---|---|---|
| 推送新代码首次部署 | 不需要手动操作 infra | 业务仓库 workflow 自动触发 |
| 回滚到某个历史版本 | 填具体的 `sha-xxxxxxxx` | 在 Docker Hub 或 Actions 日志里找 |
| 重新部署当前版本（如改了 Secrets 需重启） | **留空** | 不拉新镜像，用服务器已有镜像重启容器 |

### Git SHA、Docker Tag、Docker Digest 的关系

Actions 日志里会同时出现两种完全不同的 SHA，容易混淆：

```
# 第一种：Git commit SHA（40位）
c97365670f388ac5260bbd6f5d852ca56c1255c7

# 第二种：Docker 镜像 Digest
sha256:316659faa664555b5eb1044ddaf8a5f4eaf1e5f28463f187d646d5dbac8e53da
```

三者的完整关系：

```
git commit sha（c9736567...）
    ↓ 用前8位命名 Docker tag
Docker tag（sha-c9736567）  ← 人类操作用这个，是可变的"名字"，指向某个镜像
    ↓ 指向
Docker 镜像内容（所有层的字节）
    ↓ 对内容做 SHA256
Docker digest（sha256:316659...）  ← 内容指纹，不可变，字节级唯一
```

**关键认知**：

- Git SHA 是代码变更的唯一编码，既可以用于 `git reset` 代码回滚，也被我们复用为镜像 tag 的命名，实现镜像回滚——本质上 tag 只是一个字符串名字
- Digest 不是 Docker Hub 生成的，而是构建时由 Docker 引擎对镜像内容计算的，推送时一起上传。同一份代码 + 同一份 Dockerfile 在任何地方构建，digest 相同（确定性构建）
- Tag 是可变的（可以重新指向另一个镜像），digest 是不可变的内容指纹

**使用场景**：
- 日常操作（部署/回滚）→ 用 tag（`sha-c9736567`），可读性好
- 安全审计/溯源 → 用 digest，字节级证明"部署了哪个镜像"
