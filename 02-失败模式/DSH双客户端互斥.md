# 失败模式：两个 DSH 桌面客户端互斥，后启动的一个起不来

| 字段 | 值 |
|---|---|
| **证据等级** | 已证实（asar 内代码逐字比对，且与运行时现象一致） |
| **适用版本** | 官方 DSH Desktop `0.2.0-rc.2` vs 社区版 DeepSeek Orb `0.1.7-rc.1` |
| **发现日期** | 2026-10-03（诊断）/ 2026-10-07（处置） |
| **复核方式** | 见文末「复现方法」 |

---

## 症状

官方桌面端 + 社区客户端（DeepSeek Orb）都装了之后，**先启动的一方正常，后启动的一方起不来**：

- 后启动的一方**连窗口都不出现**，进程直接消失；或
- 窗口出现后又弹框：

  > DeepSeek Harness 无法使用
  > 应用无法启动或已意外停止。
  > 有其他正在运行的 DSH（如其他 dsh web、桌面端），无法同时启动，请退出其他正在运行的 DSH 后重启。

用户侧的感受是「官方客户端完全无法启动、彻底失去连接」。

**关键迷惑点**：Orb 侧即使只显示一个悬浮球、没有主窗口，也同样抢占——原因见下文。

## 根因：三处「只允许一个实例」的设定被原样继承

Orb 是官方桌面端的社区 fork，三处单实例设定一字未改：

| # | 冲突点 | 具体 |
|---|---|---|
| 1 | Web 服务端口 | 两侧 host 都硬编码 `args:["--no-open","--port","19387"]` |
| 2 | Electron userData | 两侧 `package.json` 的 `name` **都是 `@deepseek-ai/dsh-desktop`** 且都无 `productName` ⇒ userData 同落 `%APPDATA%\@deepseek-ai\dsh-desktop` |
| 3 | DSH home | 两侧都没设 `DSH_HOME` ⇒ 都用 `~/.dsh` |

后果链：

- ② 让两侧的 `application.requestSingleInstanceLock()` 抢同一把锁，**输的一方直接 `application.quit()`**（所以连窗口都没有）
- ① 让两侧的 host 子进程抢同一个 `127.0.0.1:19387`，输的一方 `listen EADDRINUSE` → host 退出 → 外壳弹出上面那段文案
- ③ 让 profile / session / 锁文件互相踩

## 证据

**代码位置**：两侧同名同路径 `dsh/node_modules/@deepseek-ai/dsh-desktop-host/lib/index.js`。

官方：

```js
args: [
    "--no-open",
    "--port",
    "19387"
],
```

Orb：

```js
args: [ "--no-open", "--port", "19387" ],
```

**报错文案来源**：官方 `lib/main.js` 内

```
startupAddressInUse: "有其他正在运行的 DSH（如其他 dsh web、桌面端），无法同时启动，
请退出其他正在运行的 DSH 后重启。"
```

触发条件是错误详情匹配 `/\blisten EADDRINUSE\b/`。

**崩溃日志样例**：`%APPDATA%\@deepseek-ai\dsh-desktop\logs\crash-*-host.log`
→ `dsh desktop host exited with 4294967295`（即 -1，异常退出）。

**为什么悬浮窗也算**：Orb 悬浮球的数据流必须经 host 的 Web 服务——

```js
/** NDJSON `$events` stream the floating-ball shell already posts to. */
const DESKTOP_REMOTE_STREAM_PATH = "/.dsh/remote-stream";
```

而官方 host 里搜 `floating` / `REMOTE_STREAM` 是 **0 命中**。所以悬浮球在显示 ⇒ host 在跑 ⇒ 端口被占。

## 判别口诀

| 现象 | 对应哪一处 |
|---|---|
| 后启动的**连窗口都不出现** | ② userData 单实例锁 |
| 出现窗口、再弹「有其他正在运行的 DSH…」 | ① 端口 EADDRINUSE |

## 处置

**结论：不要试图让两者共存。**

共存需同时改三处（端口、userData、DSH home），而：

- 端口与 userData 都写死在 `app.asar` 内，改动可能撞 Electron 的 asar 完整性校验
- 社区客户端每次自动更新后，改动全部失效
- `dsh web` 本身支持 `--port 0`（让系统挑空闲端口）与 `--host`、`--trusted-host`，
  但两端都在 host 里写死 `19387`，**没有对外配置项，也没有环境变量可覆盖**

已在 2026-10-07 卸载社区客户端，回收 1010.5 MB。卸载命令取自注册表
`QuietUninstallString`：`"...\Uninstall DeepSeek Orb.exe" /currentuser /S`，约 4 秒完成。

## 复现方法

```powershell
# 看谁占着端口，拿到 PID
Get-NetTCPConnection -LocalPort 19387 -State Listen | Select-Object OwningProcess
Get-Process -Id <PID> | Select-Object Id, ProcessName, Path
```

1. 完全退出 A（含托盘），确认上面那条命令没有输出
2. 启动 B，再跑一遍 → 占用者应为 B
3. 保持 B 运行，启动 A → 观察是「窗口不出现」还是「弹出 DSH 占用提示」

## 未收敛部分

- **未验证** Electron 的 asar integrity fuse 在 Windows 下是否启用。若启用，改 `app.asar`
  会直接导致应用拒绝启动——因此没有实测任何共存改法
- **未逐一确认**机器上所有社区客户端变体是否都已停用。本机除 `AppData\Local\Programs\DeepSeek Orb`
  外还有一个独立安装 `E:\deepseek-dsh\apps\dsh-desktop`（旧的定制版），**未受影响、未卸载**
- 该结论基于对 `0.2.0-rc.2` / `0.1.7-rc.1` 两个具体构建的代码比对；
  上游若改了端口或 appId，此结论需要重新验证
