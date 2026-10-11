# 图片型 PDF 批量取字：不装 OCR 依赖的一条路

**证据等级**：已证实
**适用环境**：Windows 10/11（自带 `Windows.Media.Ocr` 中文包）+ Python 3.14 + PyMuPDF 1.28.2 + Windows PowerShell 5.1
**场景**：雨课堂等导出的课件 PDF、扫描件——正文是位图，没有文字层

---

## 1. 先判断是不是图片型

```python
from pypdf import PdfReader
print(PdfReader(path).pages[0].extract_text())
```

如果只返回页眉水印和页码（例如只有 `- 1/12页 -  《 3 机器学习三要素 》`），说明**没有文字层**。

再用 PyMuPDF 看页面真实结构：

```python
import pymupdf
doc = pymupdf.open(path)
for info in doc[0].get_image_info():
    print(info["bbox"], info["width"], info["height"])
```

看到一批 `1280x720` 的位图和它们的 bbox，说明**一张幻灯片就是一张内嵌图**。
实战里遇到的两种版式：

- 2×2 拼版：每页 4 张 1280×720 + 1 个 219×70 的 logo（页脚）
- 单张：每页 1 张 1280×720

## 2. 抽原始位图，不要渲染整页再切

```python
for info in page.get_image_info(xrefs=True):
    if info["width"] < 1000 or info["height"] < 600:   # 过滤 logo / 水印
        continue
    raw = doc.extract_image(info["xref"])
```

好处：

- 拿到的是**原生分辨率**，不含页眉页脚和「雨课堂」页码条
- 不用猜 2×2 拼版的切分位置（页边距并不均匀）
- 排序用 `(y0, x0)`，即先上后下、先左后右，与幻灯片实际页码一致

**放大 2 倍再 OCR**（PIL，`Image.LANCZOS`）。1280×720 上的中文正文只有二十来像素高，放大后识别质量明显不同。

## 3. 用系统 OCR，不装 tesseract / PaddleOCR

关键约束：**这个 WinRT 投影语法在 PowerShell 7 里会报「找不到类型」**，必须用 Windows PowerShell 5.1 执行。

```powershell
Add-Type -AssemblyName System.Runtime.WindowsRuntime
$asTaskGeneric = ([System.WindowsRuntimeSystemExtensions].GetMethods() | Where-Object {
        $_.Name -eq 'AsTask' -and $_.GetParameters().Count -eq 1 -and
        $_.GetParameters()[0].ParameterType.Name -eq 'IAsyncOperation`1' })[0]

function Await($WinRtTask, $ResultType) {
    $asTask = $asTaskGeneric.MakeGenericMethod($ResultType)
    $netTask = $asTask.Invoke($null, @($WinRtTask))
    $netTask.Wait(-1) | Out-Null
    $netTask.Result
}

$null = [Windows.Media.Ocr.OcrEngine, Windows.Foundation, ContentType = WindowsRuntime]
$lang = New-Object Windows.Globalization.Language 'zh-Hans-CN'
$engine = [Windows.Media.Ocr.OcrEngine]::TryCreateFromLanguage($lang)
```

**先查本机有哪些语言**，别假设中文可用：

```powershell
[Windows.Media.Ocr.OcrEngine]::AvailableRecognizerLanguages | ForEach-Object { $_.LanguageTag }
# 本机（中文 Windows 11）：en-US、zh-Hans-CN
```

单张图的完整调用链（异步三步，都要 `Await` 包）：

```powershell
$file    = Await ([Windows.Storage.StorageFile]::GetFileFromPathAsync($f.FullName)) ([Windows.Storage.StorageFile])
$stream  = Await ($file.OpenAsync([Windows.Storage.FileAccessMode]::Read)) ([Windows.Storage.Streams.IRandomAccessStream])
$decoder = Await ([Windows.Graphics.Imaging.BitmapDecoder]::CreateAsync($stream)) ([Windows.Graphics.Imaging.BitmapDecoder])
$bitmap  = Await ($decoder.GetSoftwareBitmapAsync()) ([Windows.Graphics.Imaging.SoftwareBitmap])
$result  = Await ($engine.RecognizeAsync($bitmap)) ([Windows.Media.Ocr.OcrResult])
```

用 `$result.Lines` 逐行取（保留换行结构），不要用 `$result.Text`（会拼成一整句）。
引擎有尺寸上限 `[Windows.Media.Ocr.OcrEngine]::MaxImageDimension`（本机 10000），图片别超。
写文件用 `[System.IO.File]::WriteAllText($path, $text, (New-Object System.Text.UTF8Encoding($false)))`，避免 PS 5.1 写 BOM。

## 4. 后处理：Windows OCR 会在汉字之间插空格

输出长这样：`成 功 创 造 人 工 智 能 会 是 人 类 历 飞 上 最 大 的 事 件`

按 CJK 区间去掉字间空格即可：

```python
CJK = r"\u4e00-\u9fff\u3000-\u303f\uff00-\uffef"
re.sub(r"(?<=[" + CJK + r"])[ \t]+(?=[" + CJK + r"])", "", text)
```

## 5. 边界：这类 OCR 能做什么、不能做什么

能：抓要点、还原课件结构、判断讲了哪些概念。

不能：逐字引用。实测错字（同一批 139 张里反复出现）：

| OCR 输出 | 实际内容 |
|---|---|
| 人类历飞上 | 人类历史上 |
| 不辛的是 | 不幸的是 |
| 机器学习顶目 | 机器学习项目 |
| 留出法 1/弘 | 留出法 1/5（大小写与数字混读） |
| 还 1 / 57 | 41/57（数字被读成汉字） |

**页码这类关键数字不要信 OCR**，用 `(PDF页序-1)*每页张数+序号` 反推更可靠。

## 复核方式

拿任意一张幻灯片 PNG 跑一遍第 3 步，看输出 txt 是否非空、中文是否可读。

## 未收敛

- 只验证过雨课堂导出的两种版式，其他 PDF 生成器未测
- 没有做准确率量化对比，不确定换 RapidOCR/PaddleOCR 能提升多少
- PS 7 报"找不到类型"的确切原因没深挖（只确认 5.1 可用）
