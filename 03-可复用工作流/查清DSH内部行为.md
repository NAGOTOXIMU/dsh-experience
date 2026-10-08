# 可复用工作流：查清 DSH 自己是怎么实现的（读打包源码）

| 字段 | 值 |
|---|---|
| **证据等级** | 已证实（2026-10-07 用此法查明 shell 解析顺序，并据此解决了真实问题） |
| **适用版本** | DSH `0.2.0-rc.2` 桌面版 |
| **发现日期** | 2026-10-07 |

---

## 什么时候用

当"DSH 为什么这样"直接决定你的下一步，而文档里没有答案时：

- 某个行为由什么条件触发（例：插件何时被判为可激活）
- 某个路径 / 顺序是怎么解析的（例：agent 的 shell 到底用哪个 exe）
- UI 里某个功能叫什么、绑了什么快捷键、有没有真的实现

**不要靠猜。** DSH 的实现就在本机，而且是可读的。

## 打包资源在哪

```
<DSH 目录>\dshdesktop\resources\app.asar               ← 约 115–121 MB，实现主体
<DSH 目录>\dshdesktop\resources\app.asar.unpacked\     ← 少数未打包的依赖
```

`app.asar` 是 Electron 的归档格式，**主体是 UTF-8 文本拼接**——
所以直接按字节搜索字符串即可，**不需要先解析 asar 的目录结构**。

## 怎么做（一条命令）

用新增的常驻工具 `<工具目录>\asar-peek.py`：

```powershell
python <工具目录>\asar-peek.py <asar> -k "关键词" --before 300 --after 800 --limit 3
python <工具目录>\asar-peek.py <asar> -k "关键词" --count-only      # 先确认存不存在
python <工具目录>\asar-peek.py <asar> -k "关键词" --save out.txt    # 内容长就别刷屏
```

它会自动合并重叠命中、给出偏移量，并限制输出长度——比 `rg` 直接刷二进制有效得多。

## 实战案例：查明"我的 shell 为什么是 PowerShell 5.1"

**问题**：工具名叫 `pwsh`，可每条命令都跑在 5.1 上。这能换掉吗？

一条命令捞出 `@deepseek-ai/dsh-pwsh-local/resolve` 的完整解析逻辑：

```js
const candidates = [join(programFiles, "PowerShell", "7", "pwsh.exe")];   // ① PS7 标准位置
for (const entry of (env.PATH ?? "").split(";")) { ... }                  // ② PATH
candidates.push(join(systemRoot, "System32", "WindowsPowerShell", "v1.0", "powershell.exe"));  // ③ 5.1 兜底
```

**据此得到的结论（直接决定行动）**：

1. **PS7 是第一候选，5.1 是最后兜底** → 装 PS7 就能换掉。
2. 解析是**纯函数、每次 spawn 时求值**（不是启动时缓存）→ 理论上不必等重启；
   但 host 进程的 PATH 可能是启动快照，**实测仍需重启客户端才生效**。
3. 同一段源码的注释还说明它会处理 Store/MSIX 的 **App Execution Alias**
   （`lstat` 看到的是 symlink）→ 所以用 `winget install Microsoft.PowerShell`
   装的 MSIX 版本（per-user、免管理员，落在 `%LOCALAPPDATA%\Microsoft\WindowsApps`）
   是**被支持**的路径。

一条命令的产出，抵得上一轮试错。

## 配套工具

| 工具 | 用途 | 何时用 |
|---|---|---|
| `<工具目录>\asar-peek.py` | 在 asar 里找字符串 + 看上下文 | 想知道 DSH 内部逻辑/判据 |
| `<工具目录>\xrun.py` | 结构化执行命令（argv 数组 → JSON） | 任何原本要走 shell 字符串的命令 |

**先看这两个工具能不能用，再决定要不要现写脚本。**

## 局限（诚实标注）

- asar 里是**源码文本但没有行号**，长上下文要靠 `--before/--after` 反复调。
- 只覆盖打包进 asar 的部分；原生模块（`.node`）读不出逻辑。
- 读的是**当前安装版本**的行为，官方升级后结论可能变化——引用时**记上版本号**。
- 有些结论仍须运行时验证（例如"解析是每次执行的"≠"重启一定不必要"，
  后者被实测推翻过一次）。
