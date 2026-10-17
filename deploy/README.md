# 部署到自己的主机

把 God's Eye View 打包成 Docker 镜像，在任意一台装了 Docker 的主机上用复杂域名访问。
**除容器本身外不引入任何组件**：没有数据库、没有 Redis、没有 Nginx，应用自带的 Vite 服务就是全部。

镜像地址：`ghcr.io/workbuddy-repo/gods-eye-view:latest`

---

## 1. 让 GitHub 打包

镜像由 `.github/workflows/docker.yml` 自动构建：

- 往 `main` 推代码 → 自动构建并推送 `latest` 和 `<commit-sha>` 两个标签
- 手动触发：仓库 → **Actions** → **docker** → **Run workflow**
  - `platforms`：`linux/amd64`（默认，普通 VPS/云主机）、`linux/arm64`（Oracle ARM、树莓派）、或 `linux/amd64,linux/arm64`（同时支持两种）
  - `tag`：可选的附加标签，比如 `stable`

构建完在 Actions 的 job summary 里能看到镜像地址和 digest。

> arm64 走 QEMU 模拟，比 amd64 慢很多（十几分钟起）。知道自己主机架构的话只选一个。

## 2. 把镜像设为私有（推荐）

这个 fork 是公开仓库，首次推送后 GHCR 包默认跟随仓库可见性 = **公开**。既然只有你一个人用：

1. 打开 `https://github.com/workbuddy-repo?tab=packages`（或个人信息页 → Packages）→ `gods-eye-view`
2. **Package settings** → 最下方 Danger Zone → **Change visibility** → Private

设成私有后，主机上需要先登录才能拉（用同一个 Personal Access Token，勾选 `read:packages` 即可）：

```bash
echo <你的PAT> | docker login ghcr.io -u workbuddy-repo --password-stdin
```

## 3. 在目标主机上部署

只需要这两个文件，拷到主机任意目录（例如 `/opt/gods-eye-view/`）：

```
compose.yaml
.env            # 由 .env.example 复制而来
```

```bash
cp .env.example .env
chmod 600 .env
vi .env          # 至少把 GEV_ALLOWED_HOSTS 改成你的域名

docker compose pull
docker compose up -d
docker compose logs -f     # 看到 "[Docker] Ready at http://localhost:4173/" 即就绪
```

浏览器打开 `http://<你的域名>`。

**不用 compose 也行**，一条 `docker run`：

```bash
docker run -d --name gods-eye-view --restart unless-stopped \
  -p 80:4173 \
  --env-file /opt/gods-eye-view/.env \
  -e GEV_KEY_SETUP_TRUSTED_PEERS=none \
  -v gev-cache:/app/.gev-cache \
  -v gev-logs:/app/.gev-logs \
  ghcr.io/workbuddy-repo/gods-eye-view:latest
```

> 缓存和日志在命名卷里，`docker compose down` / 重建容器都不会丢。
> 要彻底重置：`docker compose down -v`。

## 4. 域名

把域名的 A 记录指向主机公网 IP 即可，不需要 CDN 或反代。

- 应用只认 `GEV_ALLOWED_HOSTS` 里登记过的域名，其它 Host 头一律 403 `Blocked request`。
  **改域名必须同步改这个变量并重启容器。**
- 想换端口：`.env` 里写 `GEV_PORT=8080`，然后 `docker compose up -d`。

### 想要 HTTPS（可选，多一个容器）

域名 + 80 端口的明文 HTTP 能用，但语音控制（麦克风）和浏览器定位在非 HTTPS 下会被浏览器拦掉。
要 HTTPS 最简单的办法是加一个 Caddy 容器（一个二进制，自动续签证书），把 `compose.yaml` 改成：

```yaml
services:
  gods-eye-view:
    # ... 上面的内容不变，只删掉 ports 段（不再直接对外暴露）
    # ports:
    #   - "${GEV_PORT:-80}:4173"

  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config

volumes:
  caddy-data:
  caddy-config:
```

同目录放一个 `Caddyfile`：

```
your-long-random-domain.example.com {
    reverse_proxy gods-eye-view:4173
}
```

Caddy 会自动申请 Let's Encrypt 证书，之后 `https://<域名>` 直接可用（WebSocket / HMR 它默认就转发）。

## 5. 安全：这几条请务必看

这个服务**会代你调用所有已配置的 API key**。端口一旦对公网开放：

1. **IP 直访绕不过 Host 白名单。** 代码里 IP 地址形式的 Host 默认放行，所以"域名够复杂"挡不住有人扫到你的 IP 后用 `http://<你的IP>` 访问。
   **用防火墙只放行你自己的出口 IP**，例如：
   ```bash
   ufw allow from <你的IP> to any port 80
   ufw deny 80
   ```
   更彻底的做法是干脆不暴露公网：用 Tailscale / WireGuard 组网，域名解析到内网地址。
2. **不要开放 Provider Settings 面板。** `GEV_KEY_SETUP_TRUSTED_PEERS=none` 已经写死在 compose 里，
   面板只认本机回环，外部请求一律拒绝 —— 也就是说 **key 只能写进 `.env`**，不要在网页面板里填。
3. **给 key 配额和账单告警。** 应用自带的限流是每 IP 的软限流，不是账单上限。
   Google Cloud → 账单 → 预算和告警；OpenAI → Settings → Limits。
4. **`GEV_ALLOWED_HOSTS` 不要用通配**（代码会忽略带 `*` 或前导点的条目）。
5. `.env` 里全是明文密钥，权限保持 `600`，别进 Git（`.gitignore` 已排除）。

## 6. 升级

GitHub 上重新跑一次工作流（或推代码触发）后，在主机上：

```bash
docker compose pull
docker compose up -d
```

镜像带了 `<commit-sha>` 标签，想回滚就把 `compose.yaml` 的 `image:` 改成
`ghcr.io/workbuddy-repo/gods-eye-view:<旧的sha>` 再 `up -d`。

## 7. 排错

| 现象 | 原因 |
| --- | --- |
| `Blocked request. This host is not allowed.` | `GEV_ALLOWED_HOSTS` 没写或写错域名，改完重启容器 |
| 页面能开，POWER UP 面板保存 key 报 403/500 | 预期行为：公网部署下面板禁用，把 key 写进 `.env` |
| `port is already allocated` | 主机 80 被占用，改 `GEV_PORT` |
| 日志里 `... is owned by ... but the container runs as ...` | 只在绑定源码目录时会出现；本部署方式用命名卷，不涉及 |
| 启动时报 `No default gateway` | 无害。那只是说面板不信任网关地址（本部署本就设为 none） |
