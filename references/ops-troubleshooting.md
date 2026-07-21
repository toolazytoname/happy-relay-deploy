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

## 识别陌生账号蹭中继

Happy 无密码注册,任何人知道地址都能用自己的设备注册。审计方法：

```bash
journalctl -u happy-server | grep "auth request" | grep -o "publicKey hex: [A-F0-9]*" | sort | uniq -c | sort -rn
```

对照自己各设备的公钥（首次配对时日志里出现过）。出现陌生 key 即说明有第三方在用,可 ufw/安全组封来源 IP,或换域名/端口。

## heredoc 陷阱

写 Caddyfile 含 bcrypt 哈希（`$2a$...`）时,heredoc 必须用引号界定符 `<<'EOF'` 防止 `$` 被 shell 展开吃掉。
