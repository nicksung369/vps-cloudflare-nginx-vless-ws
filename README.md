# vps-cloudflare-nginx-vless-ws

> Codex + Claude Code skill for deploying an Xray VLESS WebSocket node behind Cloudflare and nginx, with Origin CA, Full strict TLS, local-only Xray, random WebSocket path, mihomo YAML, UFW checks, and verification.

[English](#english) · [中文](#中文)

---

## English

### What it solves

This skill turns a domain-fronted Ubuntu VPS into a Cloudflare proxied VLESS WebSocket node. Clients connect to your domain over HTTPS, nginx handles TLS and WebSocket upgrade, and Xray listens only on `127.0.0.1`.

It is the domain/CDN version of a self-hosted proxy node: more moving parts than a direct REALITY node, but better origin-IP hygiene when configured correctly.

Use only on domains and servers you own or are authorized to administer, and only where self-hosted proxy services are lawful and allowed by your provider.

### When the agent should use this

Use it when you ask Codex or Claude Code to:

- deploy VLESS WebSocket behind Cloudflare
- configure nginx as a WebSocket reverse proxy to local Xray
- install Cloudflare Origin CA certs on the VPS
- generate a random WebSocket path and mihomo YAML
- audit origin exposure, UFW, nginx, and Xray binding

### What's covered

1. Cloudflare DNS and Origin CA preflight
2. Xray and nginx installation
3. Random localhost Xray port and random WebSocket path
4. Xray local-only VLESS-WS inbound
5. nginx HTTPS reverse proxy
6. UFW checks
7. mihomo YAML
8. Verification and common failure notes

### Install

```bash
# Codex
mkdir -p ~/.codex/skills
cp -r vps-cloudflare-nginx-vless-ws ~/.codex/skills/

# Claude Code
mkdir -p ~/.claude/skills
cp -r vps-cloudflare-nginx-vless-ws ~/.claude/skills/
```

Invoke explicitly:

```text
Use $vps-cloudflare-nginx-vless-ws to deploy a Cloudflare proxied VLESS WebSocket node for DOMAIN=node.example.com.
```

Claude Code can also use `/vps-cloudflare-nginx-vless-ws` when installed in its skill directory.

### Scope

- Ubuntu 22.04 / 24.04
- Cloudflare proxied DNS record
- nginx HTTPS reverse proxy
- Xray VLESS WebSocket inbound bound to localhost
- Single domain, single inbound

### License

MIT. See [LICENSE](LICENSE).

---

## 中文

### 解决什么

这个 skill 把一台带域名的 Ubuntu VPS 部署成 Cloudflare 代理后的 VLESS WebSocket 节点。客户端连你的域名和 HTTPS，nginx 负责 TLS 和 WebSocket 升级，Xray 只监听 `127.0.0.1`。

它是自建代理节点的域名/CDN 版本：比直连 REALITY 节点复杂，但配置正确时更有利于隐藏源站 IP。

请只在你拥有或被授权管理的域名和服务器上使用，并确保自建代理服务符合当地法律和服务商条款。

### 何时使用

当你让 Codex 或 Claude Code 做这些事时使用：

- 在 Cloudflare 后部署 VLESS WebSocket
- 配置 nginx 反代 WebSocket 到本机 Xray
- 在 VPS 上安装 Cloudflare Origin CA 证书
- 生成随机 WebSocket 路径和 mihomo YAML
- 审计源站暴露、UFW、nginx 和 Xray 监听地址

### 涵盖内容

1. Cloudflare DNS 和 Origin CA 预检
2. 安装 Xray 和 nginx
3. 随机本地 Xray 端口和随机 WebSocket 路径
4. Xray 仅监听本机的 VLESS-WS inbound
5. nginx HTTPS 反向代理
6. UFW 检查
7. mihomo YAML
8. 验证命令和常见坑

### 安装

```bash
# Codex
mkdir -p ~/.codex/skills
cp -r vps-cloudflare-nginx-vless-ws ~/.codex/skills/

# Claude Code
mkdir -p ~/.claude/skills
cp -r vps-cloudflare-nginx-vless-ws ~/.claude/skills/
```

显式调用示例：

```text
Use $vps-cloudflare-nginx-vless-ws to deploy a Cloudflare proxied VLESS WebSocket node for DOMAIN=node.example.com.
```

装到 Claude Code 的 skill 目录后，也可以用 `/vps-cloudflare-nginx-vless-ws` 调用。

### 适用范围

- Ubuntu 22.04 / 24.04
- Cloudflare 代理 DNS 记录
- nginx HTTPS 反向代理
- Xray VLESS WebSocket inbound 只监听本机
- 单域名、单 inbound

### 许可

MIT，见 [LICENSE](LICENSE)。
