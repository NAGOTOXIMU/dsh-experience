# 失败模式：Windows 下 `process.env.HOME` 为空导致插件全盘读不到文件

| 字段 | 值 |
|---|---|
| **证据等级** | 已证实（判别性探测 + 修复方案） |
| **适用** | Windows + 任何用 `process.env.HOME` 拼路径的 DSH 插件 |
| **实例** | `@jipika/dsh-memory` v0.6.0 |
| **发现日期** | 2026-10-06 |

---

## 症状

插件**激活正常**（`plugin_manager` 报 `fiberPhase: active`），HTTP 路由也正常响应，
但它对**明明存在的文件**报"不存在"：

```
GET /dsh-memory/content?target=rules
→ {name:"AGENTS.md", exists:false, text:"", path:"/.dsh/AGENTS.md"}
```

而 `C:\Users\<你>\.dsh\AGENTS.md` **实际有 13573 字节**。

自定义的 `~/.dsh/memory/topics/*.md` 分片也不被收录。

## 根因

插件在**模块加载时**执行：

```javascript
const HOME = process.env.HOME ?? "";
// 之后所有路径都由它拼：
const TOPICS_DIR = `${HOME}/.dsh/memory/topics`;
const RULES_DIR  = `${HOME}/.dsh`;
```

**Windows 默认没有 `HOME` 环境变量**（那是 Linux/macOS 惯例；Windows 用 `USERPROFILE`）。
于是 `HOME === ""`，路径退化成 `"/.dsh/AGENTS.md"` —— 一个从盘根开始的路径，永远不存在。

这就是 `path` 字段显示 `/.dsh/AGENTS.md`（而不是 `C:/Users/.../.dsh/AGENTS.md`）的原因。

## 判别性探测（一眼区分两种假设）

```
# 若 HOME 正常，下面应返回 AGENTS.md 的真实内容
GET http://127.0.0.1:<host端口>/dsh-memory/content?target=rules
→ exists=false 且 path=/.dsh/AGENTS.md   ⇒ HOME 为空（本缺陷）
→ exists=true  且 text 非空               ⇒ HOME 正常，问题在别处
```

**为什么选 `target=rules`**：它读 `~/.dsh/AGENTS.md`，而这个文件**一定存在**，
所以结果没有歧义。

## 处置：设用户级 `HOME`，然后重启（**不用改插件**）

```powershell
# 只设当前用户，不碰系统级
[Environment]::SetEnvironmentVariable('HOME', 'C:\Users\<你>', 'User')
# 验证已持久化
(Get-ItemProperty 'HKCU:\Environment' -Name HOME).HOME
```

**然后必须重启 DSH 桌面应用** —— 因为 `HOME` 是**模块加载时求值的常量**，
热重载不会重新求值。

**撤销**：

```powershell
[Environment]::SetEnvironmentVariable('HOME', $null, 'User')
```

### 为什么这是"干净"的修法

| 选择 | 代价 |
|---|---|
| ✅ 设环境变量 | 不改代码，插件升级不受影响；只影响当前用户；一条命令可撤销 |
| ❌ 改 `node_modules` 里的插件源码 | 插件一升级就被覆盖；改第三方代码有许可与维护问题 |
| ❌ 自写替代插件 | 本插件有 175 项断言 + 完整设置面板，重写成本极高 |

**副作用评估**：`HOME` 是跨平台工具的通用惯例，设置它对多数程序无害。
不宜再设 `USERPROFILE`（Windows 已正确提供，覆盖有风险）。

## 复现方法

```powershell
# 1) 确认 HOME 在宿主进程里为空（当前会话与宿主可能不同）
[Environment]::GetEnvironmentVariable('HOME','User')   # 用户级是否设置
$env:HOME                                              # 当前进程是否继承

# 2) 判别性探测（见上）
# 3) 跑一键验证脚本
& '<工具目录>\verify-dsh-memory.ps1'
```

## 反例 / 易混淆点

- **插件 `fiberPhase: active` 不代表它能读文件。** 本缺陷里插件全程 active、路由正常，
  功能却是坏的。
- **HTTP 路由能响应不代表路径对。** `/dsh-memory/settings` 返回 200 且值正常，
  因为它不碰文件系统。
- **`pnpm` 装的包没问题，`import` 也没问题** —— 这是纯运行时路径解析缺陷。

## 通用的教训（值得推广）

> **跨平台插件在 Windows 上失败，优先怀疑 `process.env.HOME`。**

同类高危写法：`process.env.HOME`、`~` 展开、`os.homedir()` 之外的手工拼接。
Windows 上正确的做法是 `os.homedir()`（它会读 `USERPROFILE`），
或 `process.env.USERPROFILE ?? process.env.HOME`。

## 时效性警告

若插件作者改为 `os.homedir()`，本缺陷自动消失，**那条 `HOME` 环境变量可以撤销**。
复核方式：删掉用户级 `HOME`、重启、跑上面的判别性探测；若 `exists=true` 即已修复。
