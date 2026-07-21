---
name: happy-relay-deploy
description: 自托管 Happy 中继（happy-server-light）的完整部署与维护，实现手机 App 远程控制任意多台机器上的 Claude Code / Codex。适用场景：(1) 自建/迁移 Happy relay server（公网 VPS 或 tailnet 内）；(2) Happy App 会话列表正常但点进去无限 loading（v3 接口 404 兼容问题）；(3) 中国大陆机房 + 未备案域名需要 HTTPS 的绕行部署（非标端口 + DNS-01 泛域名证书）；(4) 排查 happy auth 失败、Caddy 反代 502、中继安全加固。触发词：happy、happy-server、happy relay、中继、手机控制 claude code、happy-server-light。
---

# Happy 自托管中继部署

目标：一个中继 + 多台工作机 + 手机 App，手机任意网络可用，端到端加密。

## 核心认知（先读）

- Happy 是星型架构：中继只转发加密消息。工作机主动 outbound 连中继，无需公网 IP/端口转发。
- 账号 = 设备密钥对（无密码、无白名单）。**换中继 = 全新账号空间，历史会话不迁移**，所有客户端需重新 `happy auth` 扫码。
- iOS 拒绝明文 HTTP：中继必须有受信任 HTTPS 证书。
- **v3 兼容坑**：App v1.2+ 调用 `/v3/sessions/:id/messages`，上游 happy-server-light 仅有 v1 → 会话列表正常、点进去无限 loading。必须打补丁（见下文「v3 补丁」）。

## 决策树

```
服务器在哪?
├─ 中国大陆机房 + 域名未备案
│    → 80/443 被云厂商拦截。用非标端口(8443) + DNS-01 验证签证书
│    → DNS-01 需要 DNS 服务商 API Key(阿里云 RAM 子账号,仅 AliyunDNSFullAccess)
│    → 建议签泛域名证书 *.子域,一个 HTTPS 入口反代本机所有服务
├─ 海外机房 / 已备案域名
│    → 标准 443,Caddy 自动 HTTPS 即可
└─ 不想暴露公网
     → 各设备装 Tailscale,中继机 tailscale serve --bg 3005,地址 https://机器名.tailnet.ts.net
```

## 部署流程（公网 VPS + 非标端口为主线）

### 1. 中继本体

```bash
cd /opt && git clone --depth 1 https://github.com/leeroybrun/happy-server-light.git
cd happy-server-light && yarn install
node ./scripts/dev.mjs   # 首跑初始化 SQLite + 生成主密钥,数据在 ~/.happy/server-light/
```

**v3 补丁**（在上游合并 leeroybrun/happy-server-light#2 之前必做）：

1. 把 `assets/v3SessionRoutes.ts`（本 skill 附带，移植自官方 slopus/happy）复制到 `sources/app/api/routes/v3SessionRoutes.ts`
2. `sources/storage/seq.ts` 追加 `allocateSessionSeqBatch`（代码见 references/ops-troubleshooting.md）
3. `sources/app/api/api.ts`：import 并在 `sessionRoutes(typed);` 后加 `v3SessionRoutes(typed);`
4. `yarn build`（tsc 必须零错误）

systemd 守护：复制 `assets/happy-server.service` 到 `/etc/systemd/system/`，按实际 node 路径调整，`systemctl enable --now happy-server`，验证 `curl localhost:3005/health`。

### 2. 证书（DNS-01 + acme.sh）

```bash
curl https://get.acme.sh | sh -s email=邮箱
export Ali_Key=... Ali_Secret=...        # RAM 子账号,仅 DNS 权限
~/.acme.sh/acme.sh --issue --dns dns_ali -d 子域.域名 -d "*.子域.域名"
~/.acme.sh/acme.sh --install-cert -d 子域.域名 --ecc \
  --fullchain-file /opt/caddy/certs/happy.pem --key-file /opt/caddy/certs/happy.key \
  --reloadcmd "docker restart caddy"
```

acme.sh 自动装 cron 每天检查续期，无需人工。其他 DNS 服务商用对应 `dns_xx` 插件。

### 3. Caddy 反代（8443 统一入口）

```caddy
{
    admin off
}
happy.子域.域名:8443 {
    tls /etc/caddy/certs/happy.pem /etc/caddy/certs/happy.key
    reverse_proxy host.docker.internal:3005
}
# 其他服务照此加块,如 opencode → :8000
```

```bash
docker run -d --name caddy --restart unless-stopped -p 8443:8443 \
  --add-host host.docker.internal:host-gateway \
  -v /opt/caddy/Caddyfile:/etc/caddy/Caddyfile:ro \
  -v /opt/caddy/certs:/etc/caddy/certs:ro \
  -v /opt/caddy/data:/data -v /opt/caddy/config:/config caddy:alpine
```

改配置后 `docker restart caddy`（admin off 时 reload API 不可用）。

### 4. 防火墙（三层门,缺一不可）

1. 云厂商安全组：放行 TCP 8443（控制台操作）
2. ufw：`allow 22/tcp`、`allow 8443/tcp`、`allow in on docker0`（**漏了这条 Caddy 反代宿主机必 502**）
3. fail2ban：默认 sshd jail 即可

### 5. 客户端接入

工作机：`npm i -g happy`，`export HAPPY_SERVER_URL="https://happy.子域.域名:8443"`，`happy auth` 扫码，日常 `happy` 代替 `claude`。
手机 App：服务器地址填同一 URL（登录页右上角数据库图标），扫码配对。换过服务器的话先退出登录并删除工作机 `~/.happy/access.key`。

## 运维与排障

见 [references/ops-troubleshooting.md](references/ops-troubleshooting.md)：维护命令、症状对照表（auth 失败/Waiting 卡住/loading 转圈/502）、识别陌生账号蹭中继、备份与升级。

## 已知限制

- 语音功能自建不可用（App 写死官方 ElevenLabs agent ID，slopus/happy#472）
- 中继单点：宕机则手机端全失联，工作机本地不受影响
- 中继公开注册：任何人知道地址可蹭用,靠隐蔽性+日志审计缓解;要求高则改 tailnet-only 模式
