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

| 域名 | 容器 | 宿主机端口 |
|------|------|-----------|
| `video.daibao.site` | video-to-audio-frontend-1 | 3091 |
| `daibao.site` / `www.daibao.site` | security-quiz-game-frontend-1 | 8080 |
| `carryhub.daibao.site` | carry-hub-frontend-1 | 3081 |

---

## SSL 证书

- 工具：Let's Encrypt + Certbot
- 证书路径：`/etc/letsencrypt/live/daibao.site/`
- 有效期：90 天，Certbot 已注册 systemd timer 自动续期
- 覆盖域名：`daibao.site`、`www.daibao.site`、`video.daibao.site`

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
```

---

## 新增子域名的操作步骤

1. 在域名控制台添加 A 记录，指向 `150.158.118.89`
2. 在 `/etc/nginx/conf.d/daibao.conf` 新增一个 `server` 块（参照上方模板）
3. 测试配置：`nginx -t`
4. 重载：`systemctl reload nginx`
5. 申请证书：`certbot --nginx -d 新域名`

---

## 注意事项

- `carryhub.daibao.site` 尚未申请 SSL 证书（域名 DNS 需先添加解析后再申请）
- `security-quiz-game` 的独立容器（原来绑定宿主机 80 端口）已停止，改由宿主机 Nginx 统一接管
- 宿主机安全组只需开放 80、443，其余端口（3091、8080、3081 等）无需对公网开放
