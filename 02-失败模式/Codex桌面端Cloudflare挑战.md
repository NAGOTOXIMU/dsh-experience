# 失败模式：Codex 桌面端"发送键灰 + 无法加载工作空间设置" = Cloudflare 挑战

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（本机复现 + 官方 issue 同版本对照 + 更新后恢复） |
| **适用** | OpenAI Codex / ChatGPT 桌面端 Windows 版（本机 `26.1007.71326` 出问题；更新后恢复） |
| **发现日期** | 2026-10-15 |
| **复核方式** | 见文末"复核方法" |

---

## 症状

桌面端 composer 能输入文字，但**发送箭头是灰的、点不动**；Codex 首页显示横幅：

> 无法加载工作空间设置
> 请重试以继续聊天

英文界面为 *Couldn't load workspace settings / Retry to continue chatting.*
点"重试"无效；重启、重装、换代理都不解决。

## 根因

桌面端启动时要拉一批**账号 / 工作空间初始化接口**，而 Cloudflare 对其中几个路由返回**人机挑战**：

```
/backend-api/accounts/check/<version>   403
/backend-api/settings/user              403
/backend-api/profiles/me                403
/backend-api/me                         200   ← 对照组，正常
```

403 的响应头带 `Cf-Mitigated: challenge` 与 `Server: cloudflare`，正文是约 10 KB 的挑战页
（含 `Enable JavaScript`、`cf_chl`、`__cf`）。

**Codex 桌面端是 Rust 客户端，完不成 JS 挑战**，于是拿不到工作空间设置，
composer 的"本地配置"状态一直卡住 → 发送键永久禁用。

## 证据

**1. 客户端自己的日志**（决定性，且是唯一可靠的判据来源）

```powershell
# ts 是**秒级**时间戳，别按毫秒换算
sqlite3 "$env:USERPROFILE\.codex\logs_2.sqlite" `
  "SELECT datetime(ts,'unixepoch','localtime')||' ['||level||'] '||substr(COALESCE(feedback_log_body,''),1,220) FROM logs WHERE level IN ('ERROR','WARN') ORDER BY ts DESC LIMIT 20;"
```

故障期可见：

- `list_models{refresh_strategy=online}: failed to refresh available models: request timed out`
- `failed to connect to websocket: IO error: tls handshake eof, url: wss://chatgpt.com/backend-api/codex/responses`
- `HTTP 429`（来自插件目录 `ps/plugins/installed`、analytics 上报、Apps MCP 握手）

**2. 用客户端自己的凭据直接打那几个路由**

取 `~/.codex/auth.json` 的 `tokens.access_token`（**只用于发请求，不要打印值**）逐个打，只看状态码。

**3. 上游同版本报告**

`openai/codex` issue **#53975**：标题即
`[Windows app 26.1007.71326] Work and Codex new chats blocked by workspace settings errors; HTTP 403 cloudflare_challenge`，
症状与本机逐字一致。相关报告：`#53384`、`#53964`、`#51917`。

## 已排除的原因（都实测过，别再重复试）

| 猜测 | 实测结果 |
|---|---|
| 登录失效 / token 过期 | 否。`auth_mode=chatgpt`、access_token 在有效期内、`last_refresh` 是当天 |
| 网络断了 | 否。经代理 TLS 握手成功，WebSocket 升级请求返回 401/405（不是握手 EOF） |
| **出口 IP 被风控** | 否。把代理策略组的出口逐个换成港/日/台共 4 个节点（含两种协议），**全部 403** |
| User-Agent 被识别 | 否。伪装成 Chrome 的 UA，**仍然 403** |

## 处置

1. **更新应用到新版本 + 重启** —— 本机 2026-10-15 更新重启后恢复。
2. 更新前可临时改用**网页版**（上游 issue 里也确认 web 端正常）。
3. 这是 OpenAI 侧的问题：**不要**去改本地配置、重装、或换代理节点，都是无效功。

## ⚠️ 最关键的教训：别拿 curl 的 403 当判据

**客户端更新恢复之后，用 curl 打那三个路由依然全是 403**，而 `/backend-api/me` 始终 200。
也就是说：

> curl（以及任何非浏览器客户端）在这几个路由上会被稳定挑战。
> **它 403 只能说明"Cloudflare 挑战存在"，不能说明"Codex 现在坏着"。**

判断客户端到底好没好，**只认它自己的日志**：`logs_2.sqlite` 里还有没有 `model/list` 超时、
`thread/start` 失败 —— 恢复正常后会出现 `submission_dispatch{turn_input}`（= 消息真的发出去了）。

## 恢复后仍会看到的（属正常，别误判）

`HTTP 429` 仍会零星出现，来自 `codex_apps` MCP 握手、`ps/plugins/installed` 插件目录刷新、
analytics 上报 —— **属后台功能受限，不影响发消息**。

## 复核方法

```powershell
# 1) 先看客户端日志有没有硬故障（先看这个，别先看 curl）
sqlite3 "$env:USERPROFILE\.codex\logs_2.sqlite" "SELECT datetime(ts,'unixepoch','localtime'),level,substr(COALESCE(feedback_log_body,''),1,160) FROM logs WHERE level='ERROR' ORDER BY ts DESC LIMIT 10;"

# 2) 想看是不是 Cloudflare 挑战（注意：403 不代表客户端坏）
$t = (Get-Content "$env:USERPROFILE\.codex\auth.json" -Raw | ConvertFrom-Json).tokens.access_token
curl.exe -sS -D - -o NUL -H "Authorization: Bearer $t" "https://chatgpt.com/backend-api/settings/user" |
  Select-String 'HTTP/|Cf-Mitigated|server:'
```
