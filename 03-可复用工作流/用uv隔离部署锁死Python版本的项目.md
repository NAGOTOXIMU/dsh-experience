# 可复用工作流：用 uv 隔离部署「锁死 Python 版本」的项目

| 字段 | 值 |
|---|---|
| **验证情况** | 已证实（2026-10-16 全程实跑：`deepseekgirl` 在只有 Python 3.14 的机器上部署成功） |
| **适用环境** | Windows 11 + uv `0.12.23`；被测依赖 `wxauto4==41.1.7` |
| **用途** | 目标项目的安装脚本只认某个 Python 版本，而本机是别的版本时——不动系统 Python、不改 PATH 完成部署 |

---

## 症状

第三方项目的 `setup.bat` 里通常埋着两条路：

- 找不到指定版本 → **自动安装**（`winget install ... --silent`，失败再去 python.org
  拉安装器静默装，`PrependPath=1` → **会改用户 PATH**）；
- 以为找到 → 直接拿它去 `-m venv`。

本机可能只有 Python 3.14，用户并不想为这个项目动系统环境。

**别只看代码就断言它一定会走自动安装那条路**（我第一版就是这么写错的）：
这类「版本探测」经常坏掉，真实走向要用探针验。本例实测结果是**假阳性**——
既不装 3.12，也建不出 venv，直接在中间失败（见下面的「坑」一节）。

---

## ① 先读脚本的「找解释器」函数，不要信 README

README 写「需要 3.12」往往只是文案；真正决定行为的是脚本里的查找逻辑。
本例 `setup.bat` 的 `:find_python` 只认 3.12：

```bat
py -3.12 -c "import sys; print(sys.executable)"          &rem 没有 3.12 直接空
python -c "... print(sys.executable if sys.version_info[:2] == (3, 12) else '')"
```

两个条件只判 **主次版本号**，所以 3.14 一定被跳过 → 触发自动安装分支。

还要看两点副作用（本例都有）：

- 装完的项目 `.venv` 会被二次校验（`.venv` 不是 3.12 就直接报错退出），
  所以「手动指定 3.14 建 venv」也进不去流程；
- 有可选依赖分支（`requirements.wxauto3.txt`）失败可忽略，别被它的报错带偏。

---

## ② 一条命令事前判定「这堆依赖在该版本上到底能不能装」

不要逐个去 PyPI 猜，直接让解析器说：

```powershell
$env:HTTPS_PROXY = "http://<系统代理端口>"          # 联网前先读系统代理
uv venv   E:\Temp\check\venv314 --python 3.14
uv pip install --dry-run --python E:\Temp\check\venv314\Scripts\python.exe `
    -r requirements.windows.txt
```

`--dry-run` 只解析、不安装，失败会**直接点名是哪个包、缺哪个 ABI tag**：

```
error: No solution found when resolving dependencies
  cause: Because wxauto4==41.1.7 has no wheels with a matching Python ABI tag (e.g., `cp314`)
  hint: only found wheels for `wxauto4` with ABI tags: cp39, cp310, cp311, cp312, cp313
```

判读要点：

- **去掉那一个包再跑一次**，就知道它是唯一卡点还是第一个卡点。
  本例去掉 `wxauto4` 后，其余 22 个包（PySide6 / pywin32 / faster-whisper 及其
  ctranslate2、onnxruntime、tokenizers 链）在 3.14 上**全部解析通过**。
- 卡点包是必需还是可选，决定了「能不能硬上」：本例 `wxauto4` 是微信 4.x 后端必需，
  脚本末尾还有 `import wechatauto, wxauto4` 验证 → 3.14 此路不通。
- 顺手看 PyPI 是否有 **sdist**（源码包）；只有旧 ABI 的 wheel、又没有 sdist 时，
  升版本号也没用，这属于上游没适配。

---

## ③ 用 uv 的托管解释器部署（不碰系统 Python）

```powershell
Set-Location <项目目录>
uv venv .venv --python 3.12          # uv 自己下载一份 3.12，注册在自己的目录下
uv pip install --python .venv\Scripts\python.exe -r requirements.windows.txt
```

等效于官方脚本的安装效果，但：不改注册表、不改 PATH、`python` 仍指向原版本。

---

## ④ 按项目自己的验证清单收尾

脚本里写死的验证清单最好照抄执行（本例 6 项 import + `py_compile`）：

```powershell
& .venv\Scripts\python.exe -c "import PySide6; print(PySide6.__version__)"
& .venv\Scripts\python.exe -m py_compile main.py supervisor.py web_main.py
```

导入通过 ≠ 能跑：还要看它有没有别的启动门槛。本例机器人本体
`start.bat` 内有 `fltmc` 检查 + UAC 自提升（**需要管理员**），
而只起 Web 控制台的 `start-web.bat` 不需要——先跑不需要提权的那个验证部署。

---

## 坑与注意

- **`py -3.12` 的报错走 stdout，会让「版本探测」假阳性**（本例实测，最容易漏的一环）：
  `py` launcher 找不到指定版本时，把 `Python 3.12 not found!` 和已装版本列表
  （如 ` -3.14-64`）打到 **stdout**（不是 stderr）。而脚本里的写法是

  ```bat
  for /f "delims=" %%I in ('py -3.12 -c "import sys; print(sys.executable)" 2^>nul') do set "PYTHON_EXE=%%I"
  ```

  只丢 stderr → 报错文本被当成解释器路径赋值（`PYTHON_EXE= -3.14-64`，非空）
  → `if defined PYTHON_EXE` 判真 → **跳过自动安装**，接着拿这个假路径去
  `-m venv` 而整条流程失败。**结果是它既不装 3.12，也建不出环境。**
  现场证据：本机「系统里根本没有 3.12」，`setup.bat --check` 却返回 `CHECK_OK`。
  判别法：自己把这段探测逻辑抄成探针跑一遍；`where py` 成功且无 3.12 → 假阳性；
  `where py` 失败（PATH 里没有 py.exe）才轮到 `python -c` 分支 → 未定义 → 真会去装。
  **所以「它会自动装 Python」这句话必须加环境限定，不能一概而论。**
- **单实例端口**：本项目用 bind `127.0.0.1:47653` 做单实例锁。
  你自己起的验证实例会占住它，导致用户之后启动主程序被判「已经有一个在运行」。
  验证完先停掉自己的实例（`job_kill`），再交付。
- **记账**：这类项目依赖体积远超预期（本例 `.venv` **1.19 GB**，
  单 `pyside6-addons` 就 160 MB）。装前看一眼磁盘余量，装后报数。
- **懒加载的资源**：有的模型/权重不在源码包内（本例 faster-whisper 的 base 模型
  是首次收到语音才从 hf-mirror.com 下 ~140 MB）。别以为「放那儿就能离线用」。
- **别信「已就绪」日志**：日志里的「语音识别已就绪」只表示服务对象建好了，
  模型文件可能一个字节都没有。

## 证据

- `uv 0.12.23` 对 3.14 的解析失败/成功后重跑成功；
- PyPI `wxauto4` 文件列表：5 个 `cp39–cp313` win_amd64 wheel，无 sdist；
- 部署后 6/6 import 自检 PASS、`py_compile` 退出码 0、项目模块 import 通过。
