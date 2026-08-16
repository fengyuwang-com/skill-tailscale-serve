---
name: tailscale-serve
description: 用 Tailscale Serve 把本机 127.0.0.1 Web 服务安全暴露给手机/Tailnet 内设备访问（仅 Tailnet 内、不开 Funnel、最终入口为 https://设备名.tailnet域名/）
---

# Tailscale Serve 内网安全暴露

把运行在本机 `127.0.0.1:<port>` 的 Web 服务，通过 Tailscale Serve 封装成 Tailnet 内可达的 HTTPS 地址，供手机等 Tailnet 内设备直接访问。全程不暴露公网。

## 1. 定位与边界

- **只暴露本机 Web 服务**：源目标一律是本机 `127.0.0.1` / `localhost` 上的 HTTP 服务，不代理外部主机。
- **仅 Tailnet 内访问**：使用 `tailscale serve`（内网功能），**严禁** `tailscale funnel` 或任何公网端口映射。
- **最终入口必须是 HTTPS 域名**：`https://<设备名>.<tailnet域名>/` 形式，例如 `https://nas-work.abc123.ts.net/`。
- **禁止**用 `http://100.x.x.x:<port>` 作为最终入口：浏览器会因缺少安全上下文（secure context）导致 `crypto.randomUUID is not a function` 等 Web Crypto API 报错，部分应用直接不可用。
- 交付物是一个"手机可直接打开的 https 地址"，不是端口号或 IP。

## 2. 前置检查

```bash
# 1) 设备在线、拿到 tailnet 域名（形如 abc123.ts.net，MagicDNS 需开启）
tailscale status

# 2) 查看已有 serve 配置（重复执行前先看现状，避免覆盖误删）
tailscale serve status

# 3) 确认 HTTPS 功能可用：serve status 输出中出现 https:// 条目
#    即说明 MagicDNS + HTTPS 证书功能已启用（需要节点上已开 HTTPS，通常 tailscale up 默认开启）
tailscale serve status
```

- 若 `tailscale status` 显示设备离线或未登录：先 `tailscale up`，确认 MagicDNS 开启（`tailscale up --accept-dns`）。
- 若 `serve status` 没有任何 https:// 条目：检查 `tailscale cert <设备名>.<tailnet域名>` 能否签发证书；无法签发时先开启 HTTPS 功能再继续。

## 3. 端口盘点

先搞清本机哪些端口是真 Web 服务、哪些是无关监听，避免把 Clash 代理、VMware 后台等暴露出去。

```bash
# 列出所有 TCP 监听端口 + PID
netstat -ano | grep LISTENING

# 按 PID 查进程名（Windows）
tasklist | grep -i <PID>
```

常见无关监听（不要代理）：

- VMware：`902`(vmware-authd)、`912`(vmware-authd)
- Clash / 代理软件：`7890`、`7891`、`7897` 等
- Tailscale 自身：`41641`(UDP) 等
- 系统服务 / 数据库默认端口：视应用而定，先确认进程名与启动命令行

WSL / Docker 场景：

- WSL 里 Docker 容器映射的端口，本机看到的转发进程是 `wslrelay.exe`，此时需进 WSL 确认容器身份：
  ```bash
  wsl docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Image}}"
  ```
- 确认容器名/镜像后，再决定暴露哪个端口。

输出盘点结果：端口 → 进程名 → 是什么服务 → 是否纳入暴露。

## 4. serve 配置

### 4.1 命令语法先确认

不同版本语法有差异，先看本机版本支持什么：

```bash
tailscale serve --help
tailscale serve --https= --help   # 确认 --https 独立端口参数是否存在
```

### 4.2 路径前缀 vs 独立 HTTPS 端口（关键决策）

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 单页应用（SPA）、带 WebSocket 的终端类应用（如 xterm.js）、静态资源用**相对路径** | **独立 HTTPS 端口** `--https=<port>` | 路径前缀会改资源相对路径，导致 404 |
| 纯 API 服务（无页面、无相对路径资源） | 路径前缀 `/<prefix>` | 简单、无需记端口，入口统一为根域名 |

- 路径前缀示例：`tailscale serve --bg /api http://127.0.0.1:8000`
- 独立端口示例：`tailscale serve --bg --https=8443 http://127.0.0.1:3000`
- 多服务多端口可叠加执行，每条命令独立、互不覆盖。

### 4.3 后台模式与幂等

- 一律用 `--bg` 后台模式（前台模式会阻塞终端）。
- 重复执行相同配置是幂等的：Serve 会直接复用/覆盖同一目标，不会产生重复条目；执行前先 `tailscale serve status` 确认现状即可。
- 完成后用 `tailscale serve status` 确认生效条目。

## 5. trustedHosts 处理

先确认应用源码里是否存在 Host 校验 / allowedHosts / trustedHosts 概念，存在才改，不存在则注明"无需处理"。

```bash
# 在应用源码目录中检索
grep -rn "trustedHosts\|allowedHosts\|ALLOWED_HOSTS\|host header\|cors.*origin" --include="*.py" --include="*.js" --include="*.ts" --include="*.json" --include="*.yaml" .
```

- 找到 Host 白名单配置：把 `https://<设备名>.<tailnet域名>` 域名（不含协议更稳妥时用纯主机名）加入白名单，并让应用重载/重启。
- 未找到：在报告中注明"应用无 Host 校验，无需处理 trustedHosts"。

## 6. 验证清单

服务启动后（确认从**正确的数据/工作目录**后台启动），逐项验证：

```bash
# 1) HTTPS 证书：curl 能看到证书链、域名匹配
curl -v https://<设备名>.<tailnet域名>/ 2>&1 | grep -i "subject:\|issuer:\|SSL certificate"
#    或直接用 tailscale cert 查看签发
tailscale cert <设备名>.<tailnet域名>

# 2) 页面 200
curl -s -o /dev/null -w "%{http_code}\n" https://<设备名>.<tailnet域名>/

# 3) API 端点（如有）：替换成真实端点
curl -s -o /dev/null -w "%{http_code}\n" https://<设备名>.<tailnet域名>/api/health

# 4) WebSocket 握手：应看到 101 Switching Protocols；4xx 说明入口/代理有问题
curl --include -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  https://<设备名>.<tailnet域名>/ 2>&1 | head -20
#    有 wscat 时更直观：wscat -c wss://<设备名>.<tailnet域名>/
```

- 5) 历史工作区数据：核对应用数据目录（历史会话/工作区文件）内容仍在、未被重置（比对启动前后目录列表或文件数量）。
- 6) 服务工作目录/启动方式（Windows）：
  ```powershell
  Get-Process -Id <PID> | Select-Object ProcessName, Path
  wmic process where processid=<PID> get commandline,executablepath
  ```
  确认服务是从预期目录后台启动（如 `Start-Process` / `wsl` 内 nohup 等），而非前台临时进程。

## 7. 安全守则

- **绝不打印** API key、凭据、密码、token；日志、验证输出、报告里一律脱敏。
- 日志与错误信息用英文 ASCII 输出（避免 GBK/控制台乱码）。
- 中文文档一律 UTF-8 编码写（本 SKILL.md 即 UTF-8）。
- 报告的最终地址只给 HTTPS 域名，不给任何私密信息。

## 8. 常见坑

- **浏览器证书警告**：Tailscale 签发的证书 CA 不是系统公共 CA，手机/浏览器首次访问会告警——这是正常现象；可在受信任设备上安装 Tailscale CA 后不再告警，或临时"继续访问"。不要因此改成 http。
- **相对路径静态资源 + 路径前缀 → 404**：SPA/xterm.js 类应用必须用独立 HTTPS 端口，别走路径前缀。
- **WebSocket 不经过 serve**：老版本 Tailscale Serve 不支持 WebSocket（会 502/断开），新版（v1.5x+ 及后续）已支持；先确认版本，老版本升级后再用。
- **端口占用**：`--https=<port>` 目标端口被占用时 serve 配置不生效，先 `netstat -ano` 排查再执行。
- **`tailscale serve reset` 会清空全部 serve 配置**：慎用，只在确定要重建所有入口时使用。
- **改了 trustedHosts 没重启应用**：白名单不生效，记得重启/重载应用进程。
