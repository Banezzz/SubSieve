# SubSieve

订阅清洗网关 + 可视化管理后台，Docker Compose 部署。

订阅请求先经过黑名单、云厂商 IP 识别、UA 过滤、速率限制等多层拦截，通过后才反代到机场后端，防止订阅链接被扫描或滥用。

> 本 fork 面向 **反向代理 / Cloudflare Tunnel** 部署：TLS 在边缘（Cloudflare）终结，源站只跑明文 HTTP 并仅绑定 `127.0.0.1`，外网无法用 `IP:端口` 直连，只能经你配置的域名访问。同时修复了上游两处 bug（见 [更新日志](#更新日志)）。

---

## 部署架构

```
客户端 ──HTTPS──▶ Cloudflare Tunnel（你在 CF 配置的订阅域名）
                       │ cloudflared 跑在本机
                       ▼
            http://127.0.0.1:${GATEWAY_PORT}   ← subscribe-gateway（nginx 拦截层）
                       │ 黑名单 / 云IP / UA / 限速 / 白名单 全过后
                       ▼
                   机场后端（V2B_BACKEND）

管理后台 subscribe-admin 同样只绑 127.0.0.1:64444，经 SSH 转发
或另一条隧道 + Cloudflare Access 访问。
```

- TLS 由 Cloudflare（或其它反代）终结，源站为明文 HTTP，**不再需要证书**。
- 两个容器端口都只绑回环地址，**禁止外网 `IP:端口` 直连**。

---

## 目录结构

```
sgw/
├── .env.example            ← 配置模板（复制为 .env 使用）
├── .env                    ← 你的真实配置（已被 gitignore，勿提交）
├── docker-compose.yml      ← gateway / admin 均绑 127.0.0.1
├── setup.sh / update.sh    ← 上游脚本（本 fork 走手动 .env 部署，见下）
├── gateway/                ← nginx 拦截层（80 端口 HTTP + proxy_pass）
│   ├── nginx/nginx.conf
│   ├── nginx/subscribe_protect.conf.template
│   └── scripts/...
└── admin/                  ← PHP 管理后台（64444 端口 HTTP）
    └── src/views/dashboard.php   ← 含「订阅链接」卡片
```

---

## 前置要求

- Docker + Docker Compose
- 一个在边缘终结 TLS、回源到本机 `http://127.0.0.1:<port>` 的反代（本文以 **Cloudflare Tunnel** 为例）
- （可选，用于公网访问后台）第二条隧道域名 + Cloudflare Access

---

## 部署步骤

1. 拉取代码：

   ```bash
   git clone https://github.com/Banezzz/SubSieve.git
   cd SubSieve/sgw
   ```

2. 生成配置：

   ```bash
   cp .env.example .env
   # 生成随机后台密码与入口路径，把输出两行粘进 .env
   printf 'ADMIN_PASS=%s\nADMIN_SECRET_PATH=%s\n' \
     "$(openssl rand -base64 24 | tr -dc 'A-Za-z0-9' | head -c 20)" \
     "$(openssl rand -hex 8)"
   ```

   按注释填写 `.env`：
   - `V2B_BACKEND` / `V2B_HOST`：机场后端地址（订阅清洗后反代到此）
   - `SUBSCRIBE_PATH`：订阅路径，如 `/api/v1/client/subscribe`
   - `GATEWAY_PORT`：网关监听端口（cloudflared 回源到此，如 `47744`）
   - `GATEWAY_PUBLIC_URL`：反代后客户端用的公网订阅域名（后台展示用，可留空）
   - `AXISNOW_TRUSTED_IPS` / `REAL_IP_HEADER`：见 [真实客户端 IP](#真实客户端-ip)

3. 启动：

   ```bash
   docker compose up -d --build
   ```

4. 在 Cloudflare 配置隧道 ingress：你的订阅域名 → `http://localhost:<GATEWAY_PORT>`。

---

## 食用方法

把原订阅链接中的**域名**替换为反代后的域名，`token` 保留自己的：

```
# 原订阅链接
https://panel.airport.com/api/v1/client/subscribe?token=xxxxxxxxxxxx

# 经 Cloudflare Tunnel（HTTPS，无端口）
https://sub.your-domain.com/api/v1/client/subscribe?token=xxxxxxxxxxxx
```

> 后台「系统设置 → 订阅链接」卡片可直接生成：粘贴 **token 或整条原始订阅链接**，自动提取 token 拼出反代后的完整链接，还能一键生成二维码。

---

## 真实客户端 IP

经 Cloudflare Tunnel 回源时，cloudflared 从本机连入，容器看到的源 IP 是 **docker 网桥网关（172.x）**，不是客户端真实 IP。必须信任回源链路并读取 Cloudflare 的真实 IP 头，否则日志、云 IP 识别、黑白名单、速率限制全部按网桥 IP 生效，拦截层形同虚设。

`.env` 关键项（模板已含推荐值）：

```env
# 信任本机回源 + docker 网桥段（经发布端口回源时源 IP 被 NAT 成 172.x）
AXISNOW_TRUSTED_IPS=127.0.0.1,::1,172.16.0.0/12,10.0.0.0/8,192.168.0.0/16
# Cloudflare 用 CF-Connecting-IP；其它反代按需改 X-Forwarded-For / X-Real-IP
REAL_IP_HEADER=CF-Connecting-IP
```

> 安全性：网关端口只绑 `127.0.0.1`，外网无法直连源站伪造该头；Cloudflare 也会在隧道入口权威覆盖 `CF-Connecting-IP`。

验证：

```bash
docker exec subscribe-gateway cat /etc/nginx/subscribe/real_ip.conf
docker exec subscribe-gateway tail -n 10 /var/log/subscribe/access.log   # 第一列应为真实客户端 IP
```

---

## 管理后台

后台只绑 `127.0.0.1:64444`，两种访问方式：

- **本机 / SSH 转发**：`ssh -L 64444:127.0.0.1:64444 user@host`，然后打开 `http://localhost:64444/<ADMIN_SECRET_PATH>`
- **公网（推荐配 Cloudflare Access）**：另起一条隧道域名 → `http://localhost:64444`，并在 Zero Trust → Access 给该域名加身份策略；访问 `https://<后台域名>/<ADMIN_SECRET_PATH>`

账号密码见 `.env`（`ADMIN_USER` / `ADMIN_PASS`），入口路径为 `ADMIN_SECRET_PATH`。登录后可在「登录凭证」里改密码（存于 `admin_settings.json`，覆盖 `.env`）。

> ⚠ 后台是 PHP 管理面，公网暴露务必套 Cloudflare Access；订阅域名**不要**加 Access，否则客户端拉不到订阅。

---

## 后台功能

| 选项卡 | 功能 |
|--------|------|
| 日志 | 今日/全部切换，按 IP / 状态码 / Token 过滤，Token 全文 + 复制，一键封禁 IP，清理 7 日前旧日志 |
| 分析 | Top10 IP、Top10 Token（复制）、可疑 UA（一键封禁） |
| 封禁UA | 增删自定义封禁 UA 关键词，立即生效 |
| 白名单 | 增删、导入白名单 IP，立即生效；白名单跳过所有拦截 |
| 黑名单 | 增删黑名单 IP（deny 444），立即生效 |
| Token黑名单 | 封禁指定 Token，命中 403，支持备注 |
| 设置 | 订阅链接卡片（token / 链接提取 + 二维码）、机场上游 / 订阅路径 / 网关端口、登录凭证、界面标题 |

---

## 拦截层说明

订阅请求按以下顺序过滤，全部通过后才反代到机场后端：

1. **黑名单**：精确 IP 封禁，`deny` 返回 444
2. **云厂商 IP**：识别阿里云 / 腾讯云 / 字节 / 华为云 / UCloud / Azure / DigitalOcean / Vultr / Google / AWS 等，返回 403
3. **可疑 UA**：空 UA、curl、wget、python、Go、Java 等爬虫特征，返回 403
4. **自定义封禁 UA**：后台添加的 UA 关键词，返回 403
5. **Token 黑名单**：命中返回 403
6. **速率限制**：每分钟 20 次，burst 5，超出 429
7. **白名单**：白名单 IP 跳过上述所有拦截

云厂商 IP 库每周自动更新。

> 反代到上游前会清空客户端的 `CF-Connecting-IP` 头：当机场后端自身也套 Cloudflare 时，透传该头会触发上游 `error 1000` 导致订阅失败。网关本地仍用它做 real_ip，仅不转发给上游。

> 经反代访问域名根路径 `/` 等非订阅路径会返回 502（源站 `return 444` 断连被反代解释为 502），属正常拒绝，不影响订阅路径。

---

## 后续更新

```bash
cd SubSieve/sgw
git pull
docker compose up -d --build
```

> 本 fork 相对上游有改动（HTTP 源站、回环绑定、两个 bugfix、后台卡片等），`git pull` 可能冲突；更新前建议 `git stash`，并留意被覆盖的本地改动。

---

## 常用命令

```bash
docker compose ps
docker logs -f subscribe-gateway     # 第一列 = 真实客户端 IP
docker logs -f subscribe-admin
docker exec subscribe-gateway /scripts/update_cloud_geo.sh   # 立即刷新云 IP 库
docker compose up -d --build         # 改配置后重建
```

---

## 更新日志

- **2026-07-17（本 fork）**
  - 新增 **反向代理 / Cloudflare Tunnel** 部署模式：源站明文 HTTP、端口只绑 `127.0.0.1`、TLS 由边缘终结。
  - fix：修正 AWS `ip_prefix` 提取正则（允许冒号后空白），此前 AWS IP 段一条都没写入、完全不拦截。
  - fix：反代到上游前清空 `CF-Connecting-IP`，修复机场后端套 Cloudflare 时的 `error 1000`。
  - feat：后台「订阅链接」卡片——粘贴 token 或原始订阅链接自动提取并生成完整链接 + 二维码；session cookie 加固（HttpOnly / SameSite / 条件 Secure）。
  - 新增 `.env.example` 与 `.gitignore`。
- 2026-06-13：修复存储型 XSS 与 Nginx 配置注入两处高危漏洞，加固后台输入过滤与转义。
