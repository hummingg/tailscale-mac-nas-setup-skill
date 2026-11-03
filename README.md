# tailscale-mac-nas-setup-skill

在 macOS 与 NAS（Linux/UGOS）上安装并组网 Tailscale 的标准流程 Skill。

## 用途

把 mac 和 NAS 加入同一 Tailscale tailnet，互相用 `100.x.x.x` 直连，不受 NAT / 跨网段限制。组网完成后，远程管理交给 [`nas-ssh-ops`](https://github.com/hummingg/nas-ssh-ops-skill) skill（走 Tailscale IP SSH）。

当前拓扑（参考）：

| 节点 | Tailscale IP | 系统 | 备注 |
| ---- | ------------ | ---- | ---- |
| nas  | 100.101.102.103 | Linux (UGOS) | 绿联 DH4300 Plus |
| mac-client | 100.101.102.104 | macOS | 日常使用端 |
| mac-server | 100.101.102.105 | macOS | Mac mini |

> 示意 IP（Tailscale CGNAT 段内非真实示例），实际以 `tailscale status` 为准。

## 核心规范

- mac 走 `brew install tailscale` + GUI 浏览器登录
- NAS 走 docker 容器 + **Auth Key** 免交互登录（NAS 没浏览器，不能扫码）
- state 目录持久化，容器重建不丢身份
- 容器名 `tailscale`（不是 `tailscaled`）
- socket 路径 `/tmp/tailscaled.sock`，所有 `tailscale` 命令带 `--socket`
- 不代持 Tailscale 账号 / Auth Key / NAS sudo 密码
- Auth Key 当密码对待，不贴聊天、不写仓库

## 安装

把 `SKILL.md` 放到对应 Agent 的 skills 目录：

```bash
# Claude Code
mkdir -p ~/.claude/skills/tailscale-mac-nas-setup
ln -s "$PWD/SKILL.md" ~/.claude/skills/tailscale-mac-nas-setup/SKILL.md

# Codex
mkdir -p ~/.codex/skills/tailscale-mac-nas-setup
ln -s "$PWD/SKILL.md" ~/.codex/skills/tailscale-mac-nas-setup/SKILL.md

# WorkBuddy
mkdir -p ~/.workbuddy/skills/tailscale-mac-nas-setup
ln -s "$PWD/SKILL.md" ~/.workbuddy/skills/tailscale-mac-nas-setup/SKILL.md
```

## 使用

在 Agent 会话中说「把 mac 和 NAS 组网 Tailscale」或「NAS 上装 Tailscale 容器」即可触发。Agent 会按本规范：生成 Auth Key → mac brew 装 → NAS docker 容器 + Auth Key 登录 → 验证 100.x.x.x 互通。

## 内容

- `SKILL.md` — 规范主体（8 章：前置 / Auth Key / mac 装 / NAS 装 / 组网验证 / 维护 / 排障 / 安全纪律）

## 相关

- [nas-ssh-ops-skill](https://github.com/hummingg/nas-ssh-ops-skill) — 组网完成后的 SSH 远程管理规范
- [tailmesh-harmony-install-skill](https://github.com/hummingg/tailmesh-harmony-install-skill) — 鸿蒙手机装 TailMesh（Tailscale 兼容客户端）

## License

MIT