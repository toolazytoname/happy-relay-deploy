---
name: happy-relay-deploy
description: 自托管和维护可选的远端 Happy relay（happy-server-light），默认通过私有 tailnet 让手机 App 控制工作机。适用于跨设备远程访问、迁移 relay、修复 v3 会话接口 404、排查 Happy auth/daemon/反代故障和服务端安全。手机本地 Happy server/daemon 不需要此 skill，应使用 android-ai-stack。触发词：happy relay、远端中继、happy-server-light、手机远程控制 claude code。
---

# Happy 自托管中继部署

目标：一个私有中继 + 多台工作机 + 手机 App，默认通过 tailnet 到达。

## 核心认知（先读）

- Happy 是星型架构：中继只转发加密消息。工作机主动 outbound 连中继，无需公网 IP/端口转发。
- 账号 = 设备密钥对（无密码、无白名单）。**换中继 = 全新账号空间，历史会话不迁移**，所有客户端需重新 `happy auth` 扫码。
- iOS 拒绝明文 HTTP：中继必须有受信任 HTTPS 证书。
- **v3 兼容坑**：App v1.2+ 调用 `/v3/sessions/:id/messages`，上游 happy-server-light 仅有 v1 → 会话列表正常、点进去无限 loading。必须打补丁（见下文「v3 补丁」）。
- 手机 App、server、daemon、Claude 都在同一台 Android 上时，停止本流程并使用 `android-ai-stack`。远端 relay 不是本地方案的先决条件。
- 当前 fork 允许知道地址的人创建自己的账号空间。域名或非标端口不是访问控制；默认不得直接暴露公网。

## 决策树

```
是否真的需要跨设备远程中继?
├─ 否，server/daemon/App 都在手机 → 使用 android-ai-stack，结束
└─ 是
   ├─ 手机和工作机能加入同一 tailnet → 推荐：loopback server + Tailscale Serve
   └─ 必须公开 Internet
      → 先接受开放注册与暴露面风险；使用专用主机、TLS、备份和额外访问控制
      → 不得把“换域名/非标端口/日志审计”当作防护
```

## 部署流程（私有 tailnet 主线）

### 1. 中继本体

```bash
useradd --system --create-home --shell /usr/sbin/nologin happy
cd /opt && git clone https://github.com/leeroybrun/happy-server-light.git
cd happy-server-light
git fetch origin pull/2/head
git checkout --detach 8cfa49a74a28dbf436b3efd1728109367637c0a6
git apply /path/to/happy-relay-deploy/assets/loopback-bind.patch
chown -R happy:happy /opt/happy-server-light
runuser -u happy -- env HOME=/home/happy yarn install --frozen-lockfile
runuser -u happy -- env HOME=/home/happy node ./scripts/dev.mjs
# 健康检查通过后 Ctrl-C；数据位于 /home/happy/.happy/server-light/
```

该 commit 是 2026-08-07 核对的 [PR #2](https://github.com/leeroybrun/happy-server-light/pull/2) head，基于 fork commit `dadcf2b640e05884ebad70bc2e34de7aaee5fc3c`，已经包含 v3 补丁。固定 commit 的 API 原本硬编码监听 `0.0.0.0`；本仓 `loopback-bind.patch` 使 systemd 中的 `HAPPY_BIND_HOST=127.0.0.1` 真正生效。两份 patch 都必须在升级后重新审阅，不要 `git pull` 后直接重启。

**只在无法取得上述 commit 时手工补丁**：

1. 把 `assets/v3SessionRoutes.ts`（本 skill 附带，移植自官方 slopus/happy）复制到 `sources/app/api/routes/v3SessionRoutes.ts`
2. `sources/storage/seq.ts` 追加 `allocateSessionSeqBatch`（代码见 references/ops-troubleshooting.md）
3. `sources/app/api/api.ts`：import 并在 `sessionRoutes(typed);` 后加 `v3SessionRoutes(typed);`
4. `yarn build`（tsc 必须零错误）

创建专用 `happy` 用户，把源码和数据交给它。复制 `assets/happy-server.service` 到 `/etc/systemd/system/`，按实际 node 路径调整，`systemctl enable --now happy-server`。必须同时验证 `curl http://127.0.0.1:3005/health`、3005 只监听 `127.0.0.1`，并且 9090 没有 listener。模板依赖上面的 loopback patch；少任一步都不能声称是私有部署。

### 2. Tailscale Serve（推荐）

手机、工作机和中继加入同一 tailnet 后，在中继机执行：

```bash
tailscale serve --bg http://127.0.0.1:3005
tailscale serve status
```

把输出的 `https://<machine>.<tailnet>.ts.net` 填入 Happy App。不要再额外开放 3005/8443。

### 3. 公网 HTTPS（非默认）

只有在 tailnet 不适用且用户明确接受开放注册风险时，才继续公网部署。使用专用主机、标准 TLS、最小防火墙和独立备份；在增加经验证的接入控制前，不推荐公开服务。以下 DNS-01/8443 内容仅是历史兼容案例，不是安全默认。

#### 3.1 证书（DNS-01 + acme.sh）

```bash
curl https://get.acme.sh | sh -s email=邮箱
export Ali_Key=... Ali_Secret=...        # RAM 子账号,仅 DNS 权限
~/.acme.sh/acme.sh --issue --dns dns_ali -d 子域.域名 -d "*.子域.域名"
~/.acme.sh/acme.sh --install-cert -d 子域.域名 --ecc \
  --fullchain-file /opt/caddy/certs/happy.pem --key-file /opt/caddy/certs/happy.key \
  --reloadcmd "systemctl reload caddy"
```

acme.sh 自动装 cron 每天检查续期，无需人工。其他 DNS 服务商用对应 `dns_xx` 插件。

#### 3.2 Caddy 反代（8443 兼容入口）

```caddy
{
    admin off
}
happy.子域.域名:8443 {
    tls /etc/caddy/certs/happy.pem /etc/caddy/certs/happy.key
    reverse_proxy 127.0.0.1:3005
}
```

使用受支持的软件包安装原生 Caddy，使其与 loopback server 同处主机网络；固定版本并记录来源。改配置后用 `caddy validate`，再由 systemd reload。

#### 3.3 防火墙

1. 云厂商安全组：放行 TCP 8443（控制台操作）
2. ufw：仅放行管理来源的 SSH 和确需的 8443；3005 仍为 loopback
3. fail2ban：默认 sshd jail 即可

### 4. 客户端接入

工作机：`npm i -g happy`，`export HAPPY_SERVER_URL="https://happy.子域.域名:8443"`，`happy auth` 扫码，日常 `happy` 代替 `claude`。
手机 App：服务器地址填同一 URL（登录页右上角数据库图标），扫码配对。换过服务器的话先退出登录并删除工作机 `~/.happy/access.key`。

## 运维与排障

见 [references/ops-troubleshooting.md](references/ops-troubleshooting.md)：维护命令、症状对照表（auth 失败/Waiting 卡住/loading 转圈/502/daemon 会话闪退/root 权限检查/双版本冲突/npm 静默装坏/尸体会话）、普通用户运行 claude 的建议、识别陌生账号蹭中继、备份与升级。

## 已知限制

- 语音功能自建不可用（App 写死官方 ElevenLabs agent ID，slopus/happy#472）
- 中继单点：宕机则手机端全失联，工作机本地不受影响
- 中继缺少强接入控制时不得把公开地址当安全默认；日志审计只能发现问题，不能阻止问题
