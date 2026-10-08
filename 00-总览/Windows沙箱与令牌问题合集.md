# 总览：Windows 沙箱与令牌问题合集

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（逐条标注来源；标注"未复测"的来自历史会话记录） |
| **适用版本** | DSH `0.2.0-rc.2`（desktop profile），Windows 11 Build 26200 |
| **整理日期** | 2026-10-07 |

> 这一页把散落各处的「沙箱 / 权限 / 令牌 / 完整性」问题串成一张图。
> **排查任何"写不进去 / 起不来 / 被拒绝"之前，先来这里定位是哪一层。**

---

## 一、三层模型（关键：别把三层混为一谈）

DSH 在 Windows 上的限制**来自三个互相独立的层**，症状相似但处置完全不同：

| 层 | 限制什么 | 谁能绕过 | 典型报错 |
|---|---|---|---|
| **① 文件策略**（file policy） | 文件工具（`read`/`write`/`edit`/`glob`/`grep`）能碰哪些路径 | 切成 `danger-full-access` | `[sandbox: file access denied under workspace-write mode]` |
| **② 进程沙箱**（pwsh 子进程） | 命令**能写**哪里 —— **只有工作区**，与①无关 | **无法绕过**；改用文件工具，或让用户在普通 PowerShell 里跑 | `UnauthorizedAccessException` |
| **③ NTFS 权限 + 完整性标签** | 操作系统层面：ACL 与 Mandatory Label | 提权 / 改 ACL / 改标签 | `SetNamedSecurityInfoW failed (Win32 5)`、`0x80000003` |

**三条最容易被误判的推论**：

- **"我开了完全权限，为什么还写不进去？"** → 你碰到的是 **②**，完全权限只管 **①**。
- **"命令根本没启动，是不是我命令写错了？"** → 那是 **③**（授权失败发生在命令启动之前）。
- **"报错说拒绝访问，那就是权限不够吧？"** → 可能是 **②** 的**预期行为**，不是故障。

---

## 二、问题清单

| # | 症状 | 归类 | 详情 |
|---|---|---|---|
| 1 | 任何命令都起不来，报 `SetNamedSecurityInfoW failed (Win32 5)` | ③ | [工作区权限阻塞所有命令](../02-失败模式/工作区权限阻塞所有命令.md)（含**两种形态**：能自修 / 连修复脚本都被拒） |
| 2 | 装在**工作区内**的 Electron 应用双击无反应、退出码 `0x80000003`、**无任何日志** | ③ 的副作用 | [低完整性标签污染程序](../02-失败模式/低完整性标签污染程序.md) ← **2026-10-07 真实事故，最隐蔽的一条** |
| 3 | 完全权限下 `pwsh` 仍写不了工作区外；工作区外的仓库 `git push` 失败 | ② | [pwsh 写入范围只有工作区](../02-失败模式/pwsh写入范围只有工作区.md) |
| 4 | 受限模式下 `git push` 必然失败（schannel 取不到凭证 + 命名管道被拒） | ② | [受限沙箱下 git 推送失败](../02-失败模式/受限沙箱下git推送失败.md) |
| 5 | 受限模式下 `Get-NetTCPConnection` 报 `CimException - 拒绝访问`；配了 `-ErrorAction SilentlyContinue` 会被**静默吞成空输出**（看起来像"端口没监听"） | ② | 改用原生 `netstat -ano`（不受限） |
| 6 | 受限令牌下 `curl.exe` 的 HTTPS **一律失败** | ② | 见记忆片 `pitfalls-and-methods` |
| 7 | 插件/命令行为在 MSIX 版 PowerShell 下与预期不同 | 环境 | 见下节 |

---

## 三、判别决策树（照着走，别猜）

```
操作被拒绝了？
├─ 报 [sandbox: ... denied under workspace-write] ？
│    → ① 文件策略。切 danger-full-access，或改用别的方式
│
├─ 是 pwsh 命令，报 UnauthorizedAccessException / 写不进去？
│    → ② 进程沙箱。确认目标是否在工作区内：
│         · 在工作区内 → 看 ③（可能是 ACL/完整性）
│         · 在工作区外 → 这是预期行为，改用【文件工具】或让用户在普通 PS 里跑
│
└─ 命令**根本没启动**（连只读命令都失败）？
     → ③ NTFS 权限。跑 diagnose-windows-sandbox-acl skill 或看 icacls
         · 报 SetNamedSecurityInfoW failed → 工作区缺 WRITE_DAC/WRITE_OWNER
         · 程序起不来 + 退出码 0x80000003 → 查 Mandatory Label 是否 Low
```

---

## 四、附带：两个容易踩的环境细节

### 4.1 MSIX 版 PowerShell 7 落在受保护目录

用 `winget install Microsoft.PowerShell` 装的是 **MSIX 包**（安装器 URL 指向 GitHub Releases），
落到 `C:\Program Files\WindowsApps\Microsoft.PowerShell_<版本>_x64__<哈希>\pwsh.exe`：

- 该目录 `Access is denied`（连 `icacls` 都读不动），权限由系统接管
- 通过 `%LOCALAPPDATA%\Microsoft\WindowsApps\pwsh.exe` 这个 Execution Alias 暴露给 PATH
- DSH 解析 shell 的**第一候选**是 `C:\Program Files\PowerShell\7\pwsh.exe`（**MSI 版**的位置）→
  MSIX 版只能靠 **PATH 兜底**命中

⇒ 想要"标准路径 + 权限可查 + 第一候选直接命中"，装 **MSI 版**更省事；
MSIX 版**实测能用**，但目前没有证据表明它在沙箱下有问题（注：这点未做穷尽测试）。

### 4.2 旧版客户端对插件 pending 零容忍

社区客户端 **DeepSeek Orb 0.1.7-rc.1** 会把插件的 `pending` 判为**致命错误**
（`web boot: 1 entry did not activate`）→ 直接启动失败；
而官方 **0.2.0-rc.2** 对同样配置是容忍的。详见
[插件装了不激活](../02-失败模式/插件装了不激活.md)。

---

## 五、一句话总结

> **DSH 在 Windows 上的"拒绝"分三层：文件策略管文件工具、进程沙箱管 shell 的可写范围、NTFS/完整性管操作系统层面。
> 把三层混为一谈，就会得出"权限不够"这种错结论，然后去改错地方。**
