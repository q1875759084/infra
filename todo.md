# infra 演进待办

## 跨 Compose 服务通信：共享网络（下一步优先）

**现状**：video 的 nginx 容器要访问 monitor-backend，因为不在同一个 Compose 网络，只能靠 `host-gateway:3100` 这种 IP + 端口的写法。项目变多后端口表难以维护，且存在环境分离靠端口号区分的问题。

**目标**：建立跨 Compose 的共享网络，所有容器用**容器名**互相访问，彻底消除 IP + 端口维护成本。

**实施步骤**：

1. 在 infra 根目录声明共享网络（只需创建一次）：
   ```bash
   docker network create infra-network
   ```
   或在某个基础 compose 文件里声明为 external network：
   ```yaml
   # infra/docker-compose.network.yml
   networks:
     infra-network:
       name: infra-network
       driver: bridge
   ```

2. monitor-backend 的 compose 文件加入共享网络，并指定固定容器名：
   ```yaml
   services:
     monitor-backend:
       container_name: monitor-backend-prod   # 固定容器名，其他容器用这个名字找它
       networks:
         - default          # 自己 Compose 内部网络
         - infra-network    # 跨 Compose 共享网络
   ```

3. video、security 等业务的 nginx 改用容器名访问，替换 host-gateway 写法：
   ```nginx
   # 改前
   proxy_pass http://host-gateway:3100;
   
   # 改后
   proxy_pass http://monitor-backend-prod:3100;
   ```
   对应的 compose 文件也加入 infra-network，nginx 容器才能解析这个名字。

4. 业务容器对外的 `ports` 映射（3090、3091 等）可以逐步迁移到统一入口 nginx（见下方任务）。

**收益**：
- 不再需要维护端口映射表
- `nginx.conf` 里不再有 `host-gateway`、`172.17.0.1` 等宿主机地址
- 新增项目只需加入 infra-network，不需要找一个新端口

---

## 统一入口 nginx（中期）

**现状**：服务器对外暴露多个端口（3090、3091、3080 等），由服务器上手动维护的 nginx 按域名分发。端口分配散落在各项目 compose 文件里，没有单一来源。

**目标**：infra 仓库统一管理入口 nginx，所有业务容器不再需要 `ports` 对外暴露，只有入口 nginx 暴露 80/443。

**实施步骤**：

1. 在 infra 新建 `nginx-gateway/` 目录，管理入口 nginx 的 compose 和配置：
   ```
   infra/
     nginx-gateway/
       docker-compose.yml    # 入口 nginx 容器，ports: 80:80 / 443:443
       nginx.conf            # 按域名路由到各业务容器名
   ```

2. 入口 nginx 按域名转发：
   ```nginx
   server {
       server_name video.daibao.site;
       location / { proxy_pass http://video-frontend-prod:80; }
   }
   server {
       server_name video-dev.daibao.site;
       location / { proxy_pass http://video-frontend-dev:80; }
   }
   server {
       server_name monitor.daibao.site;
       location / { proxy_pass http://monitor-frontend-prod:80; }
   }
   ```

3. 各业务 compose 文件的 `ports` 配置全部删除，改为只 `expose`。

**收益**：
- 服务器只有 80/443 对外，内部端口完全不可见
- 新增项目只加一个 `server {}` 块，不需要找端口
- 与服务器手动 nginx 解耦，部署拓扑完整记录在 infra 仓库

---

## EVOLUTION.md 补充记录

以上演进完成后，在 `EVOLUTION.md` 补充：
- 阶段九：共享网络（跨 Compose 服务发现）
- 阶段十：统一入口 nginx（端口收敛到单点）
