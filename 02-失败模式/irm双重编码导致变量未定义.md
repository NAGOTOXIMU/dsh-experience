# 失败模式：`irm <url> | iex` 把 UTF-8 脚本变成乱码，进而报"变量未定义"

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（同机同 URL 下 5.1 与 pwsh 7.6.6 对照实测 + 字节级逆转验证） |
| **适用** | Windows PowerShell **5.1**（Windows 10/11 自带）；pwsh 7.6.6 实测无此问题 |
| **发现日期** | 2026-10-09 |
| **复核方式** | 见文末"复核方法" |

---

## 症状

执行一行式安装脚本：

```powershell
irm https://cdn.deepseek.com/api-docs/codex-deepseek-setup.ps1 | iex
```

脚本跑到中途报错：

```
检索不到变量"$ModelsPathi"，因为未设置该变量。
所在位置 行:901 字符: 25
+     Write-Ok "å·²ä¿å $ModelsPathi» $FLASH_SLUG ↔ $PRO_SLUG...
+                         ~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (ModelsPathi:String) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : VariableIsUndefined
```

两个特征同时出现，缺一不可：

1. 脚本内**硬编码的中文**全部变成 `å·²ä¿å` 这种拉丁字母乱码；
2. 变量名被"粘长"（`$ModelsPath` → `$ModelsPathi`），于是报"未定义"。

脚本自己写出的落盘产物（备份清单等）同样是乱码。

## 根因：双重编码 —— UTF-8 字节被逐个当成 Latin-1 字符

链路三层：

1. **HTTP 响应头没有 charset**。实测该 CDN 返回 `Content-Type: application/octet-stream`，
   而文件本身是 UTF-8 且含中文。
2. **Windows PowerShell 5.1 的 `Invoke-RestMethod` 在响应头无 charset 时按 ISO-8859-1 解码**。
   于是 `对`（UTF-8 = `E5 AF B9`）被读成 3 个独立字符 `å`(E5) `¯`(AF) `¹`(B9)。
   **脚本一进内存就已经坏了**，与后面 `iex` 无关。
3. **乱码把变量名粘坏**。PowerShell 变量名允许 Unicode 字母，而 `å` `æ` `ç` `Â` 都是字母。
   原文 `$ModelsPath` 后面紧跟的中文/符号坏成拉丁字母后被并进变量名，
   于是 `$ModelsPathi` 与脚本真正定义的 `$ModelsPath` 不是同一个变量。

## 证据

### 对照实验（同一 URL、同一台机器、同一代理）

| | Windows PowerShell **5.1** | pwsh **7.6.6** |
|---|---|---|
| `irm` 返回类型 | System.String | System.String |
| `irm` 返回长度 | **113900**（= 响应字节数，未合并多字节） | 110504（已按 UTF-8 合并） |
| 含正常汉字 | **False** | True |
| 含 Latin-1 连续乱码 | **True**（样例 `é¡»æ`） | False |
| 对整串做「Latin-1 取字节 → UTF-8 解码」 | **恢复出「必须恰好包含」** | — |

### 字节级验证（脚本写出的落盘产物）

`--- 对 config.toml 的改动 ---` 这一行在文件里的原始字节：

```
2D 2D 2D 20 | C3 A5 C2 AF C2 B9 | 20 ...
"--- "      |    å ¯ ¹ (= 对)    | " "
```

`对` 的 UTF-8 是 `E5 AF B9`，文件里却存成 `C3A5 C2AF C2B9` ——
这就是"每个 UTF-8 字节先变成一个 Latin-1 字符、再按 UTF-8 存盘"的铁证。

### 决定性判据：同一个文件里两种中文表现不同

损坏的清单文件里：

- `C:/Users/<用户名>/.codex/models.json` —— **正常**
- `改写 model: ...` —— **坏成 `æ¹å`**

路径来自系统 API（运行时取得的正确 Unicode 字符串），文案来自 HTTP 解码（已坏）。
**同一文件两种表现 ⇒ 不是"文件被错误读取"，而是"字符串在进内存时就分了两路"。**

## 处置

**A. 先落盘再执行（首选）**

```powershell
iwr <url> -OutFile "$env:TEMP\setup.ps1"
powershell -ExecutionPolicy Bypass -File "$env:TEMP\setup.ps1"
```

`-OutFile` 写的是原始字节、不做文本解码，文件保持正确 UTF-8，脚本引擎读它时按 UTF-8 解析。

**B. 强制按 UTF-8 解码内存字符串**

```powershell
iex ([System.Text.Encoding]::UTF8.GetString((iwr <url> -UseBasicParsing).RawContentStream.ToArray()))
```

**C. 换 pwsh 7 执行同一条命令**（实测 7.6.6 会正确按 UTF-8 解码，不会出此问题）

## 反例 / 易混淆点

- **不要归因于"系统区域设置"或"Beta: 使用 Unicode UTF-8"开关**：
  区域设置为简体中文(936) 时，UTF-8 中文若被 936 误解码会得到**汉字乱码**（如"宸叉洿鏂"），
  而不是 `å·²ä¿å` 这种拉丁字母形态；936 里根本没有 CP1252 参与。
- **不是"读取脚本时解码错"**：见上面"决定性判据"。
- **不是 `iex` 的锅**：字符串进 `iex` 之前就已经坏了。
- **判断 shell 版本别靠印象**：本次报错里的
  `所在位置 行:N 字符:M` + `+ CategoryInfo` 是 **5.1 专有格式**，
  pwsh 7 用的是 `Line |` 框式格式；再加上"命令记在 5.1 的历史文件里、pwsh 7 的历史文件为空"，
  三处证据才能定位到真实执行者。

## 一条更普遍的教训

"文本乱码"要先分清是**哪一层**做的解码：HTTP 客户端、脚本引擎、终端渲染。
**用字节级证据定位（`Encoding.GetBytes` 逆转 + 十六进制对照），不要凭乱码长相猜是哪国编码。**

## 未收敛部分

- ISO-8859-1 与 UTF-8 的默认分界到底落在 pwsh 哪个版本，未逐版本验证（只测了 5.1 与 7.6.6 两端）。
- CDN 当时的响应头没有留档，只有本次抓取的当前快照；若对方后来补了 `charset`，现象会自行消失。

## 复核方法

```powershell
$url = 'https://cdn.deepseek.com/api-docs/codex-deepseek-setup.ps1'

# 5.1 侧：预期 长度=字节数 / 无汉字 / 有 Latin-1 乱码 / 逆转后恢复中文
powershell -NoProfile -Command "
  `$s = irm '$url'
  `$s.Length
  [regex]::IsMatch(`$s, '[\u4e00-\u9fff]')
  [regex]::IsMatch(`$s, '[\u00A0-\u00FF]{4,20}')
  [System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::GetEncoding(28591).GetBytes(`$s)) -match '[\u4e00-\u9fff]'
"

# pwsh 7 侧：预期 长度 < 字节数 / 有汉字
pwsh -NoProfile -Command "`$s = irm '$url'; `$s.Length; [regex]::IsMatch(`$s, '[\u4e00-\u9fff]')"
```

> ⚠️ 检测 Latin-1 乱码**不能用** `[\u00C0-\u00FF]{2,}`：
> 中文字节解码后是「1 个 C0-FF + 2 个 80-BF」交替排列，永远凑不出连续两个 C0-FF。
> 要用 `[\u00A0-\u00FF]{4,20}` 才匹配得到 —— 这是本次排查中自己先踩到的坑。
