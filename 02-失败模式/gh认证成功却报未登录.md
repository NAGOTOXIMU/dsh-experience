# 失败模式：`gh` 认证成功却报"未登录"（配置目录写不出来）

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（一次真实事故，症状 / 根因 / 修法全部实测） |
| **适用版本** | DSH `0.2.0-rc.2` 桌面版，Windows 11（Build 26200），`gh` 2.102.0 |
| **发现日期** | 2026-10-08 |
| **影响面** | 在**受限文件策略**下执行 `gh auth login` 的用户（授权成功但索引写不进去） |

---

## 症状（最容易误判的一条）

`gh auth login --web` 走完设备码流程后，终端里**明明打印了成功**：

```
✓ Authentication complete.
- gh config set -h github.com git_protocol https
✓ Configured git protocol
mkdir C:\Users\<你>\AppData\Roaming\GitHub CLI: Access is denied.
```

**但紧接着** `gh auth status` 却报：

```
You are not logged into any GitHub hosts. To log in, run: gh auth login
```

⇒ 看起来像"授权失败 / 设备码过期 / 网络断了"，于是反复重新授权。
（实测：连着重授权 2 次，每次都要用户重新点一次 Authorize，**全都是白费**。）

## 根因：`gh` 登录要写**两个**位置，只成功了一个

| 写什么 | 位置 | 受限沙箱下 |
|---|---|---|
| **token** | Windows 凭据管理器（keyring），条目 `gh:<hostname>:<user>` | ✅ **写成功了** |
| **索引** | `%APPDATA%\GitHub CLI\hosts.yml`（记"哪台主机配哪个账号"） | ❌ **`mkdir` 被拒** |

token 在库里、索引不在 → `gh` 启动时读不到 `hosts.yml`，于是**认定自己没登录**。

> **顺带纠正一个流传的说法**：`gh` **并非**"不碰 Windows 凭据存储"。
> 恰恰相反，它默认就走 keyring（`gh auth status` 会显示 `(keyring)`），
> `cmdkey /list` 里能看到 `LegacyGeneric:target=gh:github.com:<user>`。
> 在受限沙箱里真正会失败的是**写配置目录**这一步，不是凭据存储。

## 判别（三条同时成立即确诊）

```powershell
gh auth status                                  # ① 报 "not logged into any GitHub hosts"
cmdkey /list | Select-String 'gh:'              # ② 却看得到 gh:<host>:<user> 条目
Test-Path "$env:APPDATA\GitHub CLI\hosts.yml"   # ③ False
```

**关键是第 ② 条**：token 已经在凭据库里，说明**授权早就成功了**——
所以**不要重新授权**。

## 修复（补索引即可，**不需要重新登录**）

```powershell
$dir = "$env:APPDATA\GitHub CLI"
New-Item -ItemType Directory -Path $dir -Force | Out-Null

@"
github.com:
    git_protocol: https
    user: <你的 GitHub 用户名>
"@ | Set-Content "$dir\hosts.yml" -Encoding UTF8

gh auth status    # 应显示：✓ Logged in to github.com account <你> (keyring)
```

实测结果：

```
github.com
  ✓ Logged in to github.com account <你> (keyring)
  - Token scopes: 'gist', 'read:org', 'repo'
```

> - 这一步要写 `%APPDATA%`（**工作区之外**）：受限策略下用**文件工具**做，
>   或把会话文件策略临时切到完全权限。
> - **`hosts.yml` 里只写 `git_protocol` + `user` 就够了**（实测）。
>   不要手写 `oauth_token:` 字段 —— token 本来就在 keyring 里，多写反而可能让它退化成明文存储。
> - 补完之后 `gh auth setup-git` 可以顺手跑一次，把 `credential.https://github.com.helper`
>   指向 `gh auth git-credential`，git push/pull 就免密了。

## 预防

- **让 `gh auth login` 在用户自己的普通终端里跑**（不经过 DSH 沙箱），一次就把两个位置写全。
- 或者：只要 `cmdkey /list` 里已经有 `gh:<host>:<user>`，就先按本文补 `hosts.yml`，别急着重授权。

## 反例 / 易混淆

- **不是**设备码过期：过期报的是 `expired_token`，而且 token **不会**出现在凭据管理器里。
- **不是**网络/代理问题：同一时刻用假 token 调 `gh api /zen` 能拿到真实的 `HTTP 401`，
  说明链路（含 TLS）是通的 —— 可用这条快速排除。
- **不是**版本问题：2.102.0 上实测。
- 与 [受限沙箱下 git 推送失败](受限沙箱下git推送失败.md) 的区别：那篇是 **TLS/命名管道**取不到凭据，
  本篇是 **TLS 和凭据都正常、只差一个索引文件**。

## 复核方法

```powershell
cmdkey /list | Select-String 'gh:'
Test-Path "$env:APPDATA\GitHub CLI\hosts.yml"
gh auth status
```
