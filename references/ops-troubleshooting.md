# 运维与故障排查

## 维护命令

```bash
systemctl restart happy-server              # 重启中继
journalctl -u happy-server -f               # 实时日志
curl https://<中继地址>/health              # 健康检查
~/.acme.sh/acme.sh --list                   # 证书与续期时间
docker restart caddy && docker logs caddy --tail 50  # Caddy
tar -czf happy-backup-$(date +%F).tar.gz ~/.happy/server-light/   # 备份(最重要)
cd /opt/happy-server-light && git pull && yarn install && systemctl restart happy-server  # 升级
```

## v3 补丁：seq.ts 需要追加的函数

```typescript
import type { Prisma } from "@prisma/client";

type SeqClient = Pick<Prisma.TransactionClient, "account" | "session">;

export async function allocateSessionSeqBatch(sessionId: string, count: number, tx?: SeqClient) {
    if (count <= 0) {
        return [] as number[];
    }
    const client = tx ?? db;
    const session = await client.session.update({
        where: { id: sessionId },
        select: { seq: true },
        data: { seq: { increment: count } }
    });
    const endSeq = session.seq;
    const startSeq = endSeq - count + 1;
    return Array.from({ length: count }, (_, index) => startSeq + index);
}
```

## 症状对照表

| 症状 | 原因与解法 |
|---|---|
| 手机 App 主界面红色错误 | 依次查：`curl <中继地址>/health`（本机+公网）、`systemctl status happy-server`、云安全组端口、手机能否解析域名 |
| `happy auth` 报 "Failed to create authentication request" | 终端 `echo $HAPPY_SERVER_URL` 为空（旧窗口没加载新 rc 文件）→ 新开窗口或 `source`；或 CLI 走了官方服务器而本机无代理 |
| 手机显示连接成功,电脑一直 Waiting for authentication | 手机 App 的服务器地址没改成自建中继,批准请求发去了官方服务器。App 退出登录 → 登录页数据库图标填自建地址 → 重扫 |
| 会话列表正常,点进去无限 loading | 服务端缺 v3 接口,日志可见大量 `GET /v3/sessions/.../messages` 404 → 打 v3 补丁（SKILL.md 第 1 节） |
| Caddy 502 | ufw 拦了 docker 网桥到宿主机的流量 → `ufw allow in on docker0` |
| TLS handshake internal error | 测试方法错误：SNI 必须是站点域名。正确测法 `curl https://域名:端口/health --resolve 域名:端口:127.0.0.1` |
| Caddyfile 改了不生效 | `admin off` 时 reload API 不可用,必须 `docker restart caddy` |
| 语音不可用 | App 写死官方 ElevenLabs agent ID,自建服务不可用（slopus/happy#472),忽略 |
| 手机创建会话报 Process exited unexpectedly（daemon 路径） | 先查 `~/.happy/logs/*daemon*.log`。两大高频原因：① daemon 由 systemd 裸启动,不加载 `.bashrc`,拉起的 claude 无 API 凭证闪退 → `ExecStart=/bin/bash -lc "/opt/node/bin/happy daemon start"`；② root + bypassPermissions 被新版 claude 拒绝（`--dangerously-skip-permissions cannot be used with root`）→ 用普通用户跑（见下「普通用户运行」）,临时可 `export IS_SANDBOX=1` |
| tmux 里 happy 反复刷 Continuing Claude session | 同上 root 检查,进程陷入崩溃重试循环。修复前启动的旧进程不会自动获得新环境变量,必须 `source ~/.bashrc` 后重启 happy |
| 修复后旧会话仍报 Process exited unexpectedly | 修复前创建的会话是"尸体",终态不可逆,App 里归档删除,认准修复后新建的会话 |
| Auth conflict: Both a token and an API key are set | `.bashrc` 等 rc 文件残留旧 `ANTHROPIC_API_KEY`,与 `ANTHROPIC_AUTH_TOKEN` 冲突 → 删除或 `unset` 其一 |
| 同是 claude 行为不一致（有的会话正常有的报错） | 双版本共存：native 安装（`~/.local/share/claude`）与 npm 全局并存,新旧版本行为不同（root 检查为新版新增）。`which claude` 确认解析路径,只保留一个 |
| npm 全局安装后 claude 命令失效/空壳 | 安装中断留残目录,重装报 EEXIST/ENOTEMPTY 静默失败 → `rm -rf` 包目录后 `npm i -g --force` 重装,装完必须 `claude --version` 验证,别信 exit code |
| 迁移用户后手机发消息无回复（无报错） | 每个项目目录首次启动 claude 会弹「信任此目录」确认框,会话卡在框上静默等待,tmux 窗格可见。每个窗格确认一次即永久记录（`~/.claude.json`）,之后不再出现 |
| 迁移 `.happy` 后中继全站 500（readonly database） | 中继数据 `/root/.happy/server-light` 被一并搬走,SQLite 无法写日志文件。中继数据必须留在 root（见下） |
| tmux 里启动的会话,手机发消息无回复（日志只有 RPC 探测） | 会话处于本地（local）模式,手机只读。启动加 `--happy-starting-mode remote`（如 `happy --model k3 --happy-starting-mode remote`）手机才能直接控制；终端随时按空格切回本地 |

## 普通用户运行 claude（强烈建议）

Happy 默认 bypassPermissions 模式,叠加 root 等于"任意命令免确认 + 最高权限"。建议建 `dev` 用户专跑 claude：

```bash
useradd -m -s /bin/bash dev
mv /root/.happy /root/.claude /root/.claude.json /root/claude-app /home/dev/
# ⚠️ 中继数据必须留在 root（happy-server 以 root 运行,搬走会导致 SQLite readonly 全站 500）：
mkdir -p /root/.happy && mv /home/dev/.happy/server-light /root/.happy/server-light
cp /root/.bashrc /home/dev/.bashrc && chown -R dev:dev /home/dev
```

- daemon systemd 单元加 `User=dev`,ExecStart 保留 `bash -lc`（加载 dev 的 rc 环境）
- tmux 会话重建：`su - dev -c "tmux new-session -d -s 名字 -c 项目目录"`
- 中继、Caddy、acme.sh 留在 root；claude/happy/tmux 全在 dev,爆炸半径锁在 dev 内
- 非 root 后 `IS_SANDBOX=1` 可移除（root 检查只拦 root）

## 识别陌生账号蹭中继

Happy 无密码注册,任何人知道地址都能用自己的设备注册。审计方法：

```bash
journalctl -u happy-server | grep "auth request" | grep -o "publicKey hex: [A-F0-9]*" | sort | uniq -c | sort -rn
```

对照自己各设备的公钥（首次配对时日志里出现过）。出现陌生 key 即说明有第三方在用,可 ufw/安全组封来源 IP,或换域名/端口。

## heredoc 陷阱

写 Caddyfile 含 bcrypt 哈希（`$2a$...`）时,heredoc 必须用引号界定符 `<<'EOF'` 防止 `$` 被 shell 展开吃掉。
