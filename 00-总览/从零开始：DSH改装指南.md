# 从零开始：把 DSH 改造成能干活的 AI Agent

> **这是什么**：一份**给别人看的**入门经验 —— DSH 从刚下载、一空二白，到能顺畅干活，
> 中间该做什么、千万别做什么、做错了会怎样。全部来自真实踩坑，不是理论。
>
> **适用**：DSH 桌面版（Windows），2026-10 时期。版本迭代快，遇到不一致以实际为准。

---

## 0. 先认清三个概念（否则后面全是雾）

| 概念 | 是什么 | 默认位置 |
|---|---|---|
| **DSH_HOME** | 所有配置、记忆、会话历史的根 | `C:\Users\<你>\.dsh` |
| **profile** | 一套"插件组合 + 配置" | `~\.dsh\profiles\<名字>\`（常用 `desktop`） |
| **工作区** | **一个会话的操作目录**（你在 GUI 里选的） | 你自己定 —— **这一步选错会出大事，见 §2** |

外加一个关键认知：

> **配置文件 ≠ 运行态。**
> 配置是**启动时读一次**的；改完不重启，就等于没改。判断"生效没有"必须看**运行时**，不能看文件。

---

## 1. 第一天就该做的五件事（按收益排序）

### ① 换掉 PowerShell 5.1，装 PowerShell 7 —— **收益最大的一件事**

**为什么重要**：DSH 在 Windows 上的唯一 shell 是 PowerShell。而系统自带的 **5.1** 有三个反复咬人的坑：

| 坑 | 表现 |
|---|---|
| 无 BOM 的中文 `.ps1` | 报**假语法错误**（按 GBK 解码导致），让人以为脚本写坏了 |
| 引号处理 | `gh api "..."` 这类参数会被拆坏，报 `accepts 1 arg(s), received N` |
| 外部命令输出编码 | UTF-16 输出被按 GBK 解 → **一片乱码**，只能再写 Python 绕开 |

**怎么做**：

```powershell
winget install Microsoft.PowerShell      # 装 7.x（MSIX，per-user，免管理员）
```

**注意**：**必须重启 DSH 客户端才生效** —— DSH 在启动时解析 shell 路径
（顺序是 `C:\Program Files\PowerShell\7\pwsh.exe` → 扫 PATH → 最后才回退 5.1）。

**小知识**：winget 装的是 **MSIX** 版，落在 `C:\Program Files\WindowsApps\`（受保护目录，`icacls` 都读不动）。
功能正常，只是**路径非标准、不好排查**。在意这点就手动装 GitHub 的 **MSI** 版。

### ② 装 git 与 gh，并登录

**为什么**：这是"让 agent 自己查资料"和"手动开网页"的分水岭。

```powershell
gh auth login        # 登录后，agent 能直接读仓库、搜索、建仓库、看 issue
```

**附带好处**：`gh` 是 Go 写的、**自带 TLS 实现**，
所以即使在受限沙箱里，`gh api` 也能用（而同场景下 `curl.exe` / `git push` 会失败 —— 原因见 `02-失败模式/受限沙箱下git推送失败.md`）。

> **⚠️ 勘误（2026-10-08 实测，DSH `0.2.0-rc.2`）**：此处原写"不走 Windows 凭据存储"**不成立**。
> `gh` 的 token **就是**存在凭据管理器里（`gh:<hostname>:<user>`，状态显示 `(keyring)`）。
> 受限沙箱里真正会失败的环节是**写配置目录 `%APPDATA%\GitHub CLI\`**，不是凭据存储。
> 后果很隐蔽：**授权已成功、token 已入库，但 `hosts.yml` 建不出来 → `gh auth status` 报"未登录"**。
> 处置见 `02-失败模式/gh认证成功却报未登录.md`。

### ③ 补齐命令行工具链

这些是"给 agent 装的手脚"，缺了它就只能靠人肉点界面：

```powershell
winget install BurntSushi.ripgrep.MSVC   # rg  —— 搜内容（比 findstr 快百倍）
winget install sharkdp.fd                # fd  —— 找文件
winget install jqlang.jq                 # jq  —— 处理 JSON
winget install MikeFarah.yq              # yq  —— 处理 YAML
winget install SQLite.SQLite             # sqlite3
# 7z：解压/压缩
```

### ④ 开启长期记忆

**没有记忆的 agent 每次会话都是失忆的。** 本机实测**唯一能激活**的长期记忆实现是：

```
@jipika/dsh-memory
```

装法（在 profile 目录里，注意那个 `--config` 参数 —— 不加会失败）：

```powershell
cd "$env:USERPROFILE\.dsh\profiles\desktop"
pnpm add @jipika/dsh-memory --config.auto-install-peers=false
```

然后在 `profiles\desktop\cordis.patch.yml` 里挂上：

```yaml
- insert:
    - id: dsh-memory
      name: '@jipika/dsh-memory'
```

记忆正文是 markdown 分片：`~\.dsh\memory\topics\*.md`。

> ⚠️ **别装 `@ningbainb` 那套**（user-scope / memory / personal-prompt）—— 它们在本机**无法激活**，
> 其中 `personal-prompt` 会永久 pending，**在旧版客户端上直接导致启动失败**。

### ⑤ 建三个仓库，职责分开（别混放）

| 仓库 | 放什么 | 本地是否保留 |
|---|---|---|
| **archive** | 一次性产物（诊断脚本、备份、日志） | 推送后**删本地**（省磁盘） |
| **experience** | 经验正文（就是本篇所在的库） | **保留**（要持续增补） |
| **refit** | **改装快照**：配置 + 记忆 + skills + 工具 + 自研插件 | 保留 |

**为什么要有 refit**：DSH 出问题时最省事的处置是"重装"，但改装散落在 6+ 个位置，
重装后靠记忆重建极痛苦。**平时快照好，出事就是一条命令的事。**

> 关键细节：**自研插件一定要连同源码一起备份** —— 它们不在 npm 上，只记录包名是**装不回来的**（这是我踩过的坑）。

---

## 2. 千万别做的四件事（都付过真实代价）

### ❌ 一、别把「会话工作区」设成 DSH 的安装目录

**后果**：DSH 会把工作区目录设成**低完整性**，这个标签沿 NTFS 继承污染整棵树；
装在里面的 Electron 应用（**包括 DSH 桌面端自己**）会**双击无反应、秒退、且不产生任何日志**。
退出码 `0x80000003`，查半天查不出原因。

**判别**：`icacls <目录> | Select-String Mandatory` —— 出现 `Low Mandatory Level` 就是中招。

**结论**：**工作区与"装着可执行程序的目录"必须分开。**
详见 `02-失败模式/低完整性标签污染程序.md`。

### ❌ 二、别同时装两个 DSH 桌面客户端

不同名字的客户端（例如官方版与社区版）可能**共用同一个 user-data 目录与端口** →
`requestSingleInstanceLock()` 互斥 → **后启动的那个连窗口都不出现**。

判别口诀：

- **连窗口都没有** → userData 单实例锁
- **窗口出来了，再弹"有其他正在运行的 DSH"** → 端口被占（`EADDRINUSE`）

### ❌ 三、别往 patch 里加"纯 host 插件"（不声明 `dsh.client`）

不激活是小事；**旧版客户端把插件的 `pending` 判为致命错误** → 启动直接失败
（`web boot: 1 entry did not activate`）。
详见 `02-失败模式/插件装了不激活.md`（含**三层判据**：有没有 `dsh.client`、`inject` 里的包在不在共享池）。

### ❌ 四、别把"改完配置文件"当成"已经生效"

配置是**启动时读一次**的。
判断生效与否，有且只有一条路：**看运行时**（插件的 `fiberPhase`、服务列表、界面实际表现）。
相关陷阱见 `02-失败模式/工具数据源不一致.md`。

---

## 3. 出问题时按这个顺序排查（能省很多时间）

```
① 命令根本起不来（连只读命令都失败）
   → 权限层。多半是工作区 ACL 授权失败：SetNamedSecurityInfoW failed (Win32 5)
   → 看 02-失败模式/工作区权限阻塞所有命令.md

② 命令能跑，但写不进去 / 报拒绝访问
   → 先问：目标在**工作区内**吗？
     · 工作区外 → 这是**预期行为**（shell 只能写工作区），不是故障；
                  改用文件工具，或让用户在他自己的终端里跑
     · 工作区内 → 往 ③ 查

③ 某个程序/应用起不来
   → 先查完整性标签（§2 一）；再查是不是两个客户端互斥（§2 二）

④ 插件装了不生效
   → 三层判据 + 看运行时 fiberPhase

⑤ 网络类报错（curl 的 HTTPS 失败等）
   → 先排除"沙箱限制"：受限令牌下 curl 用不了 Windows 凭据存储，
     换 Node.js / gh 这类自带 TLS 的实现试试
```

**总入口**：`00-总览/Windows沙箱与令牌问题合集.md`（三层模型 + 判别决策树）。

---

## 4. 让 agent 更好用的一条主线

> **把"给人用的操作"换成"给 agent 用的接口"。**

| 人类做法 | agent 做法 |
|---|---|
| 开浏览器搜插件 | `gh` 搜 topic + npm registry API |
| 网页上看仓库 | `gh api` / `gh repo view` |
| 文件管理器翻目录 | `rg` / `fd` |
| 编辑器改 JSON/YAML | `jq` / `yq` |
| 手敲命令读输出 | **argv 数组进、JSON 出**的小工具 |

详见 `03-可复用工作流/为agent选择工具而非GUI.md`。

---

