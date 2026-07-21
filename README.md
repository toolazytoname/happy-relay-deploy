# happy-relay-deploy

一个 Kimi / Claude Code skill：自托管 Happy 中继（happy-server-light），实现手机 App 远程控制任意多台机器上的 Claude Code / Codex。

## 解决什么问题

- 想用手机随时随地指挥家里/公司/云服务器上的 Claude Code，不想依赖官方公共服务器
- Happy App v1.2+ 连接自建 happy-server-light 时**会话点进去无限 loading**（`/v3/sessions/:id/messages` 404 兼容问题）——本 skill 附带完整补丁
- 中国大陆机房 + 未备案域名需要 HTTPS 的绕行方案（非标端口 + DNS-01 泛域名证书）

## 安装

把整个目录复制到 skills 目录即可：

```bash
git clone https://github.com/toolazytoname/happy-relay-deploy.git
cp -r happy-relay-deploy ~/.config/agents/skills/   # 或 ~/.kimi/skills/、~/.claude/skills/
```

重启 agent 后,说「帮我部署 happy 中继」即可触发。

## 内容

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 核心认知、决策树（大陆机房/海外/纯内网三条路线）、部署流程 |
| `assets/v3SessionRoutes.ts` | v3 消息接口补丁（移植自官方 slopus/happy,已提交上游 leeroybrun/happy-server-light#2） |
| `assets/happy-server.service` | systemd 守护单元模板 |
| `references/ops-troubleshooting.md` | 维护命令、故障症状对照表、陌生账号蹭中继的审计方法 |

## 相关

- 上游 PR: https://github.com/leeroybrun/happy-server-light/pull/2
- Happy 官方: https://github.com/slopus/happy
- happy-server-light: https://github.com/leeroybrun/happy-server-light

## License

MIT
