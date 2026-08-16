# skill-tailscale-serve

用 Tailscale Serve 把本机 `127.0.0.1` 上的 Web 服务安全暴露给手机/Tailnet 内设备访问的 Skill 封装。

## 这是什么

- 仅 Tailnet 内访问，不开 Funnel、不暴露任何公网端口。
- 最终交付一个手机可直接打开的 `https://<设备名>.<tailnet域名>/` 地址。
- 覆盖：前置检查、端口盘点、serve 配置决策（路径前缀 vs 独立 HTTPS 端口）、trustedHosts 处理、完整验证清单（证书/页面/API/WebSocket/历史数据/工作目录）、安全守则与常见坑。

## 怎么用

1. 让 AI 助手加载本 Skill（`tailscale-serve`）。
2. 提供待暴露的服务信息（端口、是否 SPA/含 WebSocket、数据目录位置）。
3. Skill 按步骤执行并在最后给出可用的 HTTPS 入口地址。

完整操作步骤见 [SKILL.md](./SKILL.md)。
