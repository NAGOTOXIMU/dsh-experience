# 失败模式：`pwsh` 子进程只能写工作区（与文件策略无关）

| 字段 | 值 |
|---|---|
| **证据等级** | 已证实（在 `danger-full-access` 策略下实测对照） |
| **适用版本** | DSH `0.2.0-rc.2` 桌面版，Windows |
| **发现日期** | 2026-10-07 |

---

## 症状

明明把**会话文件策略**调成"完全权限"（`danger-full-access`），
`pwsh` 里写工作区**之外**的路径**依然被拒**：

```
<DSH 目录>\...        OK        ← 工作区内
<工具目录>\...               DENIED    UnauthorizedAccessException
E:\DSH-Archive\...         DENIED
C:\Users\<你>\.dsh\...    DENIED
```

## 根因：两条通道，限制来源不同

| 通道 | 受什么限制 | 实测 |
|---|---|---|
| **文件工具**（`read` / `write` / `edit` / `glob` / `grep`） | 会话 **file policy** | 完全权限下能写 `~/.dsh`、`<工具目录>` ✅ |
| **`pwsh` 子进程**（跑命令、跑 git） | **会话工作区目录**（更严） | 完全权限下**仍然**只能写工作区 ❌ |

⇒ **"完全权限"解锁的是文件工具，不是 shell 的沙箱。**

## 后果（真实的卡点）

- **`git push` 推工作区外的仓库会失败**（要写该仓库的 `.git`）
- 不能用 `pwsh` 改 `~/.dsh` 下的配置/记忆、`<工具目录>` 下的脚本
- **绕过办法**：这类改动改用**文件工具**（`edit` / `write`）做 —— 它们遵循 file policy

## 反证与边界（重要）

- 在**工作区内**建 git 仓库，`git init / commit / push` 全程**不需要任何额外权限**（已实证）
  ⇒ 想让某个仓库能被助手顺畅操作，**把它放在工作区内**
- 需要跑工作区外的脚本时，让**用户在他自己的普通 PowerShell 窗口**里跑（不经过 DSH 沙箱）

## 与其他条目的关系

- 沙箱给工作区授权失败会报 `SetNamedSecurityInfoW failed (Win32 5)` → 见
  [工作区权限阻塞所有命令](工作区权限阻塞所有命令.md)
- 授权**成功**又会带来新的副作用（把工作区设成低完整性）→ 见
  [低完整性标签污染程序](低完整性标签污染程序.md)

## 复核方法

```powershell
foreach ($p in 'E:\<工作区>\.t', '<工具目录>\.t', "$env:USERPROFILE\.dsh\.t") {
  try { Set-Content -LiteralPath $p -Value x -ErrorAction Stop; Remove-Item $p -Force; "$p  OK" }
  catch { "$p  DENIED" }
}
```
