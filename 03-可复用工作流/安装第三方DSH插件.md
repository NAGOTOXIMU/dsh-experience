# 可复用工作流：安装第三方 DSH 插件

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（本流程在 2026-10-06 全程实跑过） |
| **适用版本** | DSH `0.2.0-rc.2`，Windows，desktop profile |
| **用途** | 从"我需要一个能力"到"确认它真的在工作"的完整流程 |

---

## 总览（六步，别跳步）

```
① 先查有没有现成的        → 避免重复造轮子
② 寻源（npm + GitHub）    → 找到候选
③ 评估（元数据 + 实测）    → 淘汰不能用的
④ 装（特定参数组合）       → 默认参数会失败
⑤ 挂载（抄 insert 行）     → 包自带的 cordis.patch.yml
⑥ 立刻验证（不要等重启）   → fiberPhase
```

---

## ① 先查本地有没有

```powershell
# 官方共享池（约 220 个包，很多能力其实已经有了）
Get-ChildItem 'C:\Users\<你>\.dsh\profiles\node_modules\@deepseek-ai' -Directory |
  Select-Object -ExpandProperty Name | Sort-Object

# 常驻自建工具
Get-ChildItem '<工具目录>' -File
```

**教训**：本项目里"自诊断 / 插件市场 / skill 管理"三个能力，官方共享池里**本来就有**，
只是没挂载。先查再装，能省掉大量工作。

## ② 寻源

```powershell
# npm（插件主战场）
$s = Invoke-RestMethod 'https://registry.npmjs.org/-/v1/search?text=dsh-plugin&size=25' -TimeoutSec 30
$s.objects | ForEach-Object { "{0}  v{1}  {2}" -f $_.package.name,$_.package.version,($_.package.description -replace '\s+',' ') }
```

```powershell
# GitHub 主题（注意：不要只按 star 排，该主题有 1.8 万个仓库，大量蹭标签的）
# 本机已有工具：<工具目录>\gh-dsh-search.py（按 cordis.patch.yml / dsh 字段判真伪）
python <工具目录>\gh-dsh-search.py --verify
```

**注意**：GitHub 的 `topic:dsh-plugin` 里混着简历站、低代码平台、图床等无关热门项目。
判定"是不是 DSH 插件"要用**结构性特征**（仓库里有 `cordis.patch.yml`），
而且要**递归查全树**（插件常在 `packages/` 或 `integrations/` 子目录）。

## ③ 评估（四道关）

```powershell
# 本机已有工具，可直接跑
python <工具目录>\eval-memory-candidates.py   # 改里面的 CANDIDATES 列表即可复用
python <工具目录>\check-peers.py              # peer 是否能被共享池满足
python <工具目录>\probe-memory-import.py      # 隔离实测 import
```

| 关 | 看什么 | 判定 |
|---|---|---|
| **兼容声明** | `dsh.engines.dsh` / `dsh.compatibility.dsh` | 未声明 = **未知**，不能当兼容 |
| **安装期脚本** | `scripts` 里的 `preinstall`/`install`/`postinstall`/`prepare` | 有就要 `--ignore-scripts` 或放弃 |
| **体积** | `dist.unpackedSize` | >10 MB 要问清用途 |
| **client 半边** | `dsh.client` 是否存在 | **没有 ⇒ 极可能不激活**（见失败模式） |
| **peer 可满足** | 逐个查共享池 | 缺 peer 不影响加载，但 pnpm 会去下载而失败 |

### 关键：隔离实测 import（不污染环境）

把 tarball 解包到 **`profiles\desktop` 下的临时目录**，这样 Node 能向上解析到
`profiles\node_modules\@deepseek-ai` 里的 peer，然后 `import` 入口文件：

```javascript
import { pathToFileURL } from 'node:url';
const m = await import(pathToFileURL(entryPath).href);
console.log(Object.keys(m));   // 应包含 apply / inject / name
```

**为什么必须实测**：元数据看不出版本是否真的兼容。本项目实测 5 个候选**全部 import 成功**，
推翻了"peer 范围旧 = 不能用"的猜测。

> ⚠️ **但 import 成功 ≠ 装进 profile 能激活**。前者只验证模块可加载，
> 后者还要过 Cordis 的条目激活（见 [插件装了不激活](../02-失败模式/插件装了不激活.md)）。

## ④ 安装（默认参数会失败）

```powershell
$sys=(Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings').ProxyServer
$env:HTTP_PROXY="http://$sys"; $env:HTTPS_PROXY=$env:HTTP_PROXY; $env:NO_PROXY='localhost,127.0.0.1,::1'
$node='C:\Users\<你>\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe'
$pnpm='C:\Users\<你>\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\pnpm\bin\pnpm.cjs'

& $node $pnpm add --dir 'C:\Users\<你>\.dsh\profiles\desktop' --ignore-scripts `
  --registry 'https://registry.npmjs.org/' `
  --config.auto-install-peers=false `
  --config.strict-peer-dependencies=false `
  '<包名>@<版本>'
```

| 开关 | 为什么必须 |
|---|---|
| `--config.auto-install-peers=false` | **最关键**。否则 pnpm 会去下载 peer 声明的旧版 DSH 包，报 `NO_MATCHING_VERSION` |
| `--config.strict-peer-dependencies=false` | 版本不匹配只警告，不中断 |
| `--registry https://registry.npmjs.org/` | 避免 registry 解析异常 |
| `--ignore-scripts` | 规避 `prepare` / `postinstall` 执行代码 |

**坑**：`--ignore-peer-dependencies` 和 `--legacy` **不是** pnpm 11 的合法参数，别试。

## ⑤ 挂载

包自带的 `cordis.patch.yml` 就是要抄的内容：

```powershell
Get-Content 'C:\Users\<你>\.dsh\profiles\desktop\node_modules\<包名>\cordis.patch.yml' -Raw
```

把它里面的 `- insert:` 段落**追加**到
`profiles\desktop\cordis.patch.yml` 末尾。

**注意**：patch 是**整行替换 config，不是合并**。若它覆盖了某个已有条目
（例如 `session-query-sqlite`），要意识到你会**替换掉原有 config 的全部字段**。

## ⑥ 立刻验证（不要等重启）

```
plugin_manager action=list_plugins offset=<n> limit=<m>
   → enabled 应为 true，fiberPhase 应为 active

cordis_inspect_query platform=host provider=Service method=listService
   input={"service":"<插件该注册的服务>"}
   → 报 no catalogued Service named X 即未注册
```

**若 `fiberPhase` 是 `null`**：

1. 先看该包有没有 `dsh.client` → 没有就是那条失败模式，**别改了**
2. **不要反复 `set_plugin`** —— 对这个症状无效，只会往 patch 堆无用条目
3. 换一个声明了 `dsh.client` 的同类插件

---

## 回滚

```powershell
# 卸载
& $node $pnpm remove '<包名>' --dir 'C:\Users\<你>\.dsh\profiles\desktop'
# 再从 cordis.patch.yml 里删掉对应 insert 行
```

**务必先备份** `cordis.patch.yml`：

```powershell
Copy-Item 'C:\Users\<你>\.dsh\profiles\desktop\cordis.patch.yml' `
          "E:\DSH-Archive\<任务名>-<日期>\备份\cordis.patch.yml.$(Get-Date -Format yyyyMMdd-HHmmss).bak"
```

---

## 环境相关的坑（Windows 专属）

| 坑 | 表现 | 处理 |
|---|---|---|
| **没有 `pwsh`** | `pwsh : 无法将"pwsh"项识别为...` | 用 `powershell -File` 或 `& '路径.ps1'` |
| **`.ps1` 缺 BOM** | 报 `字符串缺少终止符` / `意外的标记"}"` 这类**假语法错误** | PowerShell 5.1 按 GBK 解码无 BOM 的 UTF-8。写完必须加 BOM |
| **`$Home` 是只读自动变量** | `无法覆盖变量 Home` | 参数名改用 `$DshHome` |
| **`gh api` 引号被拆** | `accepts 1 arg(s), received N` | 用 Python `subprocess`（list 形式）或 `cmd /c` |
| **代理端口会变** | 联网超时 | 一律从注册表读 `ProxyServer`，不要硬编码 |

## 时效性警告

本流程基于 `0.2.0-rc.2`。官方若修掉"纯 host 不激活"或改变 peer 解析行为，
第 ③④⑥ 步的判定需重新核对。
