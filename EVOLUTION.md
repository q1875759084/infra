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
| `docker-compose.yml` | 服务定义、网络、Volume（环境无关） |
| `docker-compose.prod.yml` | 只写差异：`ports: 80:80` |
| `docker-compose.dev.yml` | 只写差异：`ports: 8080:80` |

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

---

## 待完成

- [ ] infra 仓库配置 Secrets
- [ ] 前端/后端仓库配置 INFRA_DEPLOY_TOKEN（需先生成 PAT）
- [ ] 联调验证整条链路
