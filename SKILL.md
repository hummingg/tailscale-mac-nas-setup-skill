---
name: tailscale-mac-nas-setup
description: 在 macOS 与 NAS（Linux/UGOS）上安装并组网 Tailscale 的标准流程。当需要把 mac 和 NAS 加入同一 tailnet、生成 Auth Key、装 Tailscale 容器/客户端、验证 100.x.x.x 互通、排查节点 offline / 登录失败 / 容器 socket 时触发。核心规范：mac 走 brew + GUI 登录、NAS 走 docker 容器 + Auth Key 免交互、state 目录持久化、容器名 tailscale、socket 路径 /tmp/tailscaled.sock 须带 --socket、不代持账号密码、组网后用 nas-ssh-ops skill 远程管理。
---

# mac + NAS 装 Tailscale 并组网

> 目标：mac 与 NAS 加入同一 tailnet，互相用 100.x.x.x 直连，不受 NAT / 跨网段限制。
> 组网完成后，远程管理交给 `nas-ssh-ops` skill（走 Tailscale IP SSH）。

## 0. 前置条件

1. 有一个 Tailscale 账号（https://login.tailscale.com，免费版够 100 设备）。
   > ⚠️ **账号选择提醒**：Tailscale 默认引导用 Google 账号登录，但国内网络访问 Google 不稳定（登录页/回调可能超时失败）。**慎重用 Google 账号**，优先选 GitHub、Microsoft 或邮箱注册的账号，登录更顺。
2. mac 能上网，NAS 能上网且已开 SSH（见 `nas-ssh-ops` 0.1）。
3. 已在 mac 配好 NAS 的 SSH 免密（见 `nas-ssh-ops` 0.2/0.3）——后面要远程给 NAS 装容器。

## 1. 生成 Auth Key（NAS 免交互登录用）

NAS 走 docker 容器，没有浏览器，必须用 **Auth Key** 认证，不能扫码登录。

在 Tailscale 控制台：**Settings → Keys → Generate auth key**

- 选 **Reusable**（可复用，避免每次重启容器都换 key）
- 选 **Ephemeral**（可选；容器重建后旧节点自动清理，避免节点堆积）
- Tags 按需（如 `tag:nas`）

复制 `tskey-auth-xxxxxxxxxxxx`，**只显示一次**，存到安全处（如 1Password）。**不要贴进聊天记录**。

## 2. mac 安装

### 2.1 装

```bash
brew install tailscale
```

### 2.2 启动并登录

```bash
sudo tailscale up
```

会打印一个 https://login.tailscale.com/... 链接，浏览器打开授权即可。**这一步必须人工点**，Agent 不代持账号。

### 2.3 验证

```bash
tailscale status
tailscale ip -4      # 应输出 100.x.x.x
```

## 3. NAS 安装（docker 容器 + Auth Key）

NAS 是 Linux（绿联 DH4300 Plus / UGOS，等价群晖 / 飞牛 / 普通 Linux），用官方 tailscale 容器。

### 3.1 准备目录与 docker 组

```bash
ssh nas 'mkdir -p /volume1/docker/tailscale/state'
ssh nas 'id | grep -q docker || sudo usermod -aG docker $USER'   # 没加组才加
```

> 加 docker 组后**新 SSH 会话自动生效**，不用重启 NAS（见 `nas-ssh-ops` §4）。
> 若没免密 sudo，让用户在 NAS 上手动执行 `sudo usermod -aG docker $USER`，**不代持密码**。

### 3.2 起容器

```bash
ssh nas 'bash -s' <<'EOF'
docker run -d \
  --name tailscale \
  --restart=unless-stopped \
  -v /volume1/docker/tailscale/state:/var/lib/tailscale \
  -v /dev/net/tun:/dev/net/tun \
  --cap-add=net_admin \
  --cap-add=sys_module \
  tailscale/tailscale:latest
EOF
```

要点：

- 容器名 **`tailscale`**（不是 `tailscaled`，后面 `tailscale` 命令靠它）；
- `state` 目录必须持久化，否则容器重建后节点身份丢失、要重新登录；
- `--cap-add=net_admin` + `/dev/net/tun` 是 WireGuard 内核接口所需。

### 3.3 用 Auth Key 登录

```bash
ssh nas 'docker exec tailscale tailscale up --authkey=tskey-auth-xxxxxxxxxxxx --hostname=nas --accept-routes'
```

- `--hostname=nas`：在 tailnet 里显示成 `nas` 而不是容器 ID；
- `--accept-routes`：接受其他节点通告的子网路由（按需）；
- **Auth Key 不写进任何持久化文件**，只作为命令参数一次性传入；容器重启后 state 里已有身份，不用再 up。

### 3.4 验证

```bash
ssh nas 'docker exec tailscale tailscale status'
ssh nas 'docker exec tailscale tailscale ip -4'    # 应输出 100.x.x.x
```

## 4. 组网验证

在 mac 上：

```bash
tailscale status                       # 应看到 nas 节点，状态 active
ping -c 3 <nas 的 100.x.x.x>           # 应通
ssh hummingg@<nas 的 100.x.x.x> 'echo OK'   # SSH 走 Tailscale，等价局域网
```

在 NAS 上（反向）：

```bash
ssh nas 'docker exec tailscale tailscale status'   # 应看到 mac 节点
ssh nas 'docker exec tailscale ping -c 3 <mac 的 100.x.x.x>'
```

三节点拓扑（参考，实际以 `tailscale status` 为准）：

| 节点 | Tailscale IP | 系统 | 备注 |
| ---- | ------------ | ---- | ---- |
| nas  | 100.101.102.103 | Linux (UGOS) | 绿联 DH4300 Plus |
| mac-client | 100.101.102.104 | macOS | 日常使用端 |
| mac-server | 100.101.102.105 | macOS | Mac mini |

> 上表为**示意 IP**（Tailscale CGNAT 段 100.64.0.0/10 内的非真实示例），实际以 `tailscale status` 输出为准，请勿照抄。

## 5. 常用维护命令

### 5.1 容器 socket 路径（踩坑点）

容器内 `tailscaled` 的 socket 在 **`/tmp/tailscaled.sock`**，不是默认的 `/var/run/tailscale/tailscaled.sock`。**所有 `tailscale` 命令都要带 `--socket`**：

```bash
ssh nas 'docker exec tailscale tailscale --socket=/tmp/tailscaled.sock status'
```

> 不带 `--socket` 会报 `dial unix /var/run/tailscale/tailscaled.sock: connect: no such file`。

### 5.2 重启 / 看日志

```bash
ssh nas 'docker restart tailscale'
ssh nas 'docker logs --tail 50 tailscale'
```

### 5.3 退出 tailnet（不删容器）

```bash
ssh nas 'docker exec tailscale tailscale down'
```

### 5.4 容器名以 `docker ps` 为准

若起容器时改了名字，后面所有 `docker exec <名字>` 同步改。先 `docker ps --filter ancestor=tailscale/tailscale` 确认。

## 6. 排障

| 现象 | 排查 |
| ---- | ---- |
| mac `tailscale up` 卡住 | 浏览器没开；复制链接手动打开授权 |
| NAS 容器起不来 | `docker logs tailscale` 看 `tun` 权限；确认 `/dev/net/tun` 存在、`--cap-add=net_admin` 加了 |
| `tailscale up --authkey` 报 `invalid key` | key 用完或过期；控制台重新生成 |
| `tailscale status` 看不到对端 | 两端不在同一 tailnet（账号不同）；或对端 `down` 了 |
| `dial unix .../tailscaled.sock: no such file` | 没带 `--socket=/tmp/tailscaled.sock`（见 §5.1） |
| ping 不通但 status 显示 active | ACL 拒绝；控制台 ACLs 加 `{"action": "accept", "src": ["*"], "dst": ["*:*"]}`（测试用） |
| 容器重建后节点变成新的 | state 目录没持久化；检查 `-v` 挂载是否生效 |

## 7. 安全与纪律

1. **Auth Key 当作密码对待**：不复盘、不贴聊天、不写仓库；用完即弃，长效 key 定期轮换。
2. **不代持 Tailscale 账号**：mac 的 `tailscale up` 浏览器授权交给用户执行。
3. **NAS sudo 同 `nas-ssh-ops`**：需要交互 sudo 的操作交给用户，不代持密码。
4. **组网后 SSH 走 Tailscale IP**：不再依赖局域网 IP，配置/脚本里把 NAS 地址换成 `100.x.x.x`。
5. **容器 state 持久化**：丢失 state 等于节点身份丢失，要重新登录并产生新节点。

## 8. 后续

组网完成后，日常 NAS 远程管理交给 **`nas-ssh-ops`** skill：把 SSH 目标从局域网 IP 换成 Tailscale IP（如 `100.101.102.103`），其余流程（heredoc 批量、脱敏、备份→sed→diff）不变。