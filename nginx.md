# 宿主机 Nginx 配置说明

## 架构概述

服务器上采用两层 Nginx 结构：

```
用户请求
    ↓
【宿主机 Nginx】 /etc/nginx/conf.d/daibao.conf
  职责：SSL 终止 + 子域名分发
  监听：80（HTTP 重定向）、443（HTTPS）
    ↓ proxy_pass 到各容器宿主机端口
【容器内 Nginx】（各项目 Docker 镜像内置）
  职责：静态资源服务 + /api/* 转发到 Node.js
    ↓
【Node.js 后端进程】
```

容器内 Nginx 收到的是已解密的明文 HTTP，不感知 SSL。

---

## 域名 → 容器端口映射

| 域名 | 容器 | 宿主机端口 | SSL |
|------|------|-----------|-----|
| `video.daibao.site` | video-to-audio-frontend-1 | 3091 | ✅ |
| `daibao.site` / `www.daibao.site` | security-quiz-game-frontend-1 | 8080 | ✅ |
| `carryhub.daibao.site` | carry-hub-frontend-1 | 3081 | ❌ |
| `cowatch.daibao.site` | cowatch-frontend-1 | 3070 | ✅ |
| `test.cowatch.daibao.site` | cowatch-frontend-1（dev） | 3071 | ✅ |

---

## SSL 证书

- 工具：Let's Encrypt + Certbot
- 证书路径：`/etc/letsencrypt/live/daibao.site/`
- 有效期：90 天，Certbot 已注册 systemd timer 自动续期
- 覆盖域名：`daibao.site`、`www.daibao.site`、`video.daibao.site`、`cowatch.daibao.site`、`test.cowatch.daibao.site`
- 最近一次扩充：2026-06-10（新增 `test.cowatch.daibao.site`）
- 有效期至：2026-09-08

续期测试：
```bash
certbot renew --dry-run
```

---

## 配置文件路径

服务器上：`/etc/nginx/conf.d/daibao.conf`

---

## 当前生效配置（含 Certbot 自动写入的 SSL 部分）

```nginx
# ============================================================
# 宿主机 Nginx 反向代理配置
# 职责：SSL 终止 + 子域名分发，不处理业务逻辑
# 各容器端口：
#   video.daibao.site     → video-to-audio-frontend-1  (3091)
#   daibao.site / www     → security-quiz-game-frontend-1 (8080)
#   carryhub.daibao.site  → carry-hub-frontend-1 (3081)
#   cowatch.daibao.site      → cowatch-frontend-1 (3070)  [生产]
#   test.cowatch.daibao.site → cowatch-frontend-1 (3071)  [测试]
# ============================================================

# video.daibao.site → video-to-audio (3091)
server {
    server_name video.daibao.site;

    location / {
        proxy_pass http://127.0.0.1:3091;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # SSE 长连接：关闭 Nginx 缓冲，否则进度事件会被积压后批量发送
    location /api/convert/progress/ {
        proxy_pass http://127.0.0.1:3091;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding on;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/daibao.site/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/daibao.site/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

# daibao.site + www.daibao.site → security-quiz-game (8080)
server {
    server_name daibao.site www.daibao.site;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/daibao.site/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/daibao.site/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

# carryhub.daibao.site → carry-hub (3081)
server {
    listen 80;
    server_name carryhub.daibao.site;

    location / {
        proxy_pass http://127.0.0.1:3081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# cowatch.daibao.site → cowatch (3070)
# WebSocket 升级头：/socket 端点必须透传 Upgrade + Connection，否则 WS 握手失败
server {
    server_name cowatch.daibao.site;

    # 宿主机 nginx 不感知业务大小限制，设 0 表示不限制，由容器内 nginx 和后端自行负责
    client_max_body_size 0;

    location /socket {
        proxy_pass http://127.0.0.1:3070;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location / {
        proxy_pass http://127.0.0.1:3070;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/daibao.site/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/daibao.site/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

# HTTP → HTTPS 重定向（由 Certbot 自动生成）
server {
    if ($host = www.daibao.site) { return 301 https://$host$request_uri; }
    if ($host = daibao.site) { return 301 https://$host$request_uri; }
    listen 80;
    server_name daibao.site www.daibao.site;
    return 404;
}
server {
    if ($host = video.daibao.site) { return 301 https://$host$request_uri; }
    listen 80;
    server_name video.daibao.site;
    return 404;
}
server {
    if ($host = cowatch.daibao.site) { return 301 https://$host$request_uri; }
    listen 80;
    server_name cowatch.daibao.site;
    return 404;
}

# test.cowatch.daibao.site → cowatch 测试环境 (3071)
# WebSocket 升级头：/socket 端点必须透传 Upgrade + Connection，否则 WS 握手失败
server {
    server_name test.cowatch.daibao.site;

    # 宿主机 nginx 不感知业务大小限制，设 0 表示不限制，由容器内 nginx 和后端自行负责
    client_max_body_size 0;

    location /socket {
        proxy_pass http://127.0.0.1:3071;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location / {
        proxy_pass http://127.0.0.1:3071;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/daibao.site/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/daibao.site/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = test.cowatch.daibao.site) { return 301 https://$host$request_uri; }
    listen 80;
    server_name test.cowatch.daibao.site;
    return 404;
}
```

---

## 新增子域名的操作步骤

1. 在域名控制台添加 A 记录，指向 `150.158.118.89`，等待 DNS 生效
2. 在 `/etc/nginx/conf.d/daibao.conf` 新增一个 HTTP `server` 块（供 Certbot HTTP-01 验证用）
3. 测试配置：`nginx -t`，重载：`systemctl reload nginx`
4. 扩充证书（将新域名追加到现有证书）：
   ```bash
   certbot certonly --nginx --expand \
     -d daibao.site -d www.daibao.site -d video.daibao.site -d cowatch.daibao.site -d test.cowatch.daibao.site \
     -d 新域名 \
     --non-interactive --agree-tos
   ```
5. 将 HTTP 临时块替换为 443 SSL 块，加上 HTTP→HTTPS 重定向块，`systemctl reload nginx`
6. 更新本文档

---

## 注意事项

- `carryhub.daibao.site` 尚未申请 SSL 证书（域名 DNS 需先添加解析后再申请）
- `security-quiz-game` 的独立容器（原来绑定宿主机 80 端口）已停止，改由宿主机 Nginx 统一接管
- 宿主机安全组只需开放 80、443，其余端口（3091、8080、3081、3070、3071 等）无需对公网开放
- `cowatch.daibao.site` / `test.cowatch.daibao.site` 的 `/socket` location 需特殊配置 WebSocket 升级头（`Upgrade` + `Connection: upgrade`），与其他普通 HTTP 服务不同
- `cowatch.daibao.site` / `test.cowatch.daibao.site` 设置 `client_max_body_size 0`（不限制），宿主机 nginx 不感知业务大小限制，由容器内 nginx（`client_max_body_size 4096M`）和后端自行负责
- `test.cowatch.daibao.site` 对应 CoWatch dev 部署（3071 端口），`cowatch.daibao.site` 对应 prod 部署（3070 端口）
