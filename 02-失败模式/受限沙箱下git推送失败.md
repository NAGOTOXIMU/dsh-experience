# 失败模式：受限沙箱下 `git push` 到 GitHub 必然失败

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（两类报错均有实测输出；放宽后同一条命令成功） |
| **适用** | Windows + DSH 受限文件策略（`workspace-write`）下的 `git push` |
| **环境** | DSH `0.2.0-rc.2` / git 2.55.0.windows.5 / gh 2.102.0 |
| **发现日期** | 2026-10-06 |
| **复核方式** | 受限模式下跑 `git ls-remote origin`；仍报下文两类错误即本条仍成立 |

---

## 症状

受限模式（`workspace-write`）下推送 GitHub，**默认 `schannel` 后端**：

```
$ git -C E:\git push -u origin main
fatal: unable to access 'https://github.com/<你的账号>/sandbox.git/':
  schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS (0x8009030e)
```

换成 OpenSSL 后端，**报错变了，但依然失败**：

```
$ git -C E:\git -c http.sslBackend=openssl ls-remote origin
      0 [main] sh (18928) ...\Git\usr\bin\sh.exe: *** fatal error -
        couldn't create signal pipe, Win32 error 5
error: failed to execute prompt script (exit code 66)
fatal: could not read Username for 'https://github.com': No such file or directory
```

**同一分钟、同一台机器上，`gh` 完全正常**：

```
$ gh api user --jq .login
<你的账号>                    # exit 0
```

## 根因

两个**互相独立**的受限令牌限制叠在一起，所以"换一个后端"只换了报错、换不来成功：

| 后端 | 被什么挡住 |
|---|---|
| `schannel`（本机系统级 `http.sslbackend` 默认值） | 受限令牌取不到 TLS 客户端凭证句柄，连握手都进不去 |
| `openssl` | TLS 过了，但**创建命名管道被拒**（`Win32 error 5`）→ git 起不了凭据助手（`gh auth git-credential` 是 shell 脚本），于是退回交互式索要用户名 → 非交互环境必然失败 |

`gh` 之所以不受影响：它是 Go 程序，用**自带的 crypto/tls**（不碰 schannel），
且认证在自己的进程内完成，不需要 git 那套 helper 管道。

## 判别性证据

三组输出出自同一次排查、同一台机器、同一分钟：

1. `git push`（schannel）→ `SEC_E_NO_CREDENTIALS`
2. `git ls-remote`（openssl）→ `couldn't create signal pipe, Win32 error 5`
3. `gh api user` → 成功

→ 可排除「网络不通 / 代理配错 / gh 未登录 / 仓库无权限」，指向**沙箱令牌限制**。

## 处置：把 `git push` 单独放宽跑一次

**不要**去改全局 git 配置（`http.sslBackend=openssl` 治不好，管道那条限制还在）。
正解是只给这一条命令放宽文件策略（一次性审批），用**默认后端、默认凭据助手**直接推：

```
（放宽模式下）git -C E:\git push -u origin main
→  * [new branch]      main -> main          # exit 0
   branch 'main' set up to track 'origin/main'.
```

推完立刻回到受限模式，无需保留放宽状态。

## 反例 / 易混淆点

- **不是代理问题**：当时 `HTTP_PROXY`/`HTTPS_PROXY` 已正确指向系统代理，`gh api` 走同一代理成功。
- **不是 gh 登录失效**：`gh auth status` 报已登录，token scope 含 `repo`。
- **不是仓库不存在**：远端仓库正是同一次 `gh repo create` 建出来的。
- **`gh repo create --source <路径> --push` 会「半成功」**：仓库建好了、`origin` 也配好了，
  只有最后一步 push 失败（`failed to run git: exit status 128`）。
  别把它当整条命令失败——先 `git remote -v` 看远端是否已配上，否则会重复建仓库。

## 时效性警告

DSH 是 preview，沙箱实现可能变化。复核方式：受限模式下跑 `git ls-remote origin`；
若不再报上述两类错误，本条应就地改标 `已推翻`（并注明限制已解除的版本）。
