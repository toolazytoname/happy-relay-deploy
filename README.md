# happy-relay-deploy

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Network default](https://img.shields.io/badge/default-tailnet--only-blue.svg)](SECURITY.md)

[简体中文](README.md) · [English](README.en.md)

一个 Claude Code / Codex skill：在确有跨设备需求时自托管 Happy 中继（happy-server-light），让手机 App 远程控制工作机。

手机本身运行 Happy server/daemon 的本地自洽方案**不需要本仓**，见
[`android-ai-stack`](https://github.com/toolazytoname/android-ai-stack)。本仓只承接可选的远端 relay 安全边界。

## 解决什么问题

- 想通过私有 tailnet 指挥家里/公司/云服务器上的 Claude Code，不想依赖官方公共服务器
- Happy App v1.2+ 连接自建 happy-server-light 时**会话点进去无限 loading**（`/v3/sessions/:id/messages` 404 兼容问题）——本 skill 附带完整补丁
- 已明确接受公开注册风险后，研究公网 HTTPS 的非默认部署路线

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
| `SKILL.md` | 核心认知、手机本地/私有 tailnet/公网风险决策树、部署流程 |
| `assets/v3SessionRoutes.ts` | v3 消息接口补丁的固定副本；优先直接使用已核对的 PR commit |
| `assets/loopback-bind.patch` | 让固定上游 commit 真正读取 `HAPPY_BIND_HOST`，避免伪 loopback 配置 |
| `assets/happy-server.service` | systemd 守护单元模板 |
| `references/ops-troubleshooting.md` | 维护命令、故障症状对照表、陌生账号蹭中继的审计方法 |

## 相关

- 上游 PR: https://github.com/leeroybrun/happy-server-light/pull/2
- Happy 官方: https://github.com/slopus/happy
- happy-server-light: https://github.com/leeroybrun/happy-server-light
- Android 本地 AI 栈: https://github.com/toolazytoname/android-ai-stack

## License

MIT

补丁来源与固定 commit 见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
