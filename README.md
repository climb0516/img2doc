# 图片转文档 · img2doc

把图片里的文字识别出来，排版成 A4 文档，可在线编辑，一键导出 PDF / Word。识别全程在浏览器本地完成，图片不会上传到任何服务器。

在线使用：https://climb0516.github.io/img2doc/

![界面截图](docs/screenshot.png)

## 功能

- **本地识别**：基于 PaddleOCR（PP-OCRv4）中英文模型，在浏览器内运行，中文识别准确率高
- **按段落还原**：同一段的多行合并成一个段落，随纸张宽度自动换行；原图段首的缩进保留为两个全角空格
- **两栏识别与排版**：自动检测两栏图片（支持拍照倾斜），左右栏分别识别后按「左栏 → 右栏」顺序输出，A4 页面、PDF、Word 均可两栏排版
- **标题自动识别**：居中的短行自动识别为标题，居中加粗显示
- **破折号修正**：自动把被误识别成「一」的「——」还原
- **A4 分页预览**：按真实 A4 尺寸分页，可直接在纸上编辑；支持缩放
- **排版调整**：单栏 / 两栏、字体（宋体 / 黑体 / 楷体）、字号、行距、上下 / 左右边距
- **导出**：导出 PDF、导出 Word（.docx，A4 纸张）、打印、复制全文
- **多图批量**：支持点击、拖拽、粘贴截图上传多张图片，可调整顺序

## 使用方法

1. **上传图片**：点击选择、拖拽，或直接 ⌘V / Ctrl+V 粘贴截图
2. **识别**：点「开始识别」。首次使用需下载约 30MB 的模型，之后浏览器会缓存
3. **排版**：调整字体、字号、行距、边距；也可以直接点击纸张修改文字
4. **导出**：右上角点「导出」，选择「导出为 PDF」或「导出为 Word」；也可以「打印」或「复制全文」

「导出为 PDF」会把每页 A4 转成图片后生成 PDF，和页面效果完全一致，但文字不能选中、搜索，每页约几百 KB。需要文字可选的 PDF 时，点「打印」，在打印窗口中目标选「存储为 PDF」，纸张 A4，边距选「无」。

## 本地运行

不需要构建，直接用 Chrome 打开 `index.html` 即可。识别模型和 PDF / Word 导出库从 jsDelivr CDN 加载，首次使用需要联网。

也可以起一个静态服务器：

```
python3 -m http.server 8000
```

然后访问 http://localhost:8000

## 实现说明

### 段落还原

PaddleOCR 返回每一行文字及其四边形坐标，在此基础上还原段落：

- **行**：纵向中心落在同一范围内的文本框合并为一行（表单），间隔超过 0.9 个字宽时补全角空格
- **缩进**：与前后各 2 行的最左边界比较，超过 0.75 个字宽算 1 个字；局部边界可以抵消拍照倾斜
- **空行**：两行间距按中位行距换算，超出一行的视为空行
- **标题**：单个文本框、不超过 20 字、缩进不少于 2 字、中心距版心中线 3 个字宽以内、不以句末标点结尾
- **分段**：遇到标题、有缩进的行或空行时开始新段落，其余行接到上一段末尾；原图段首有缩进的，段首加两个全角空格

### 分栏检测

PaddleOCR 会把左右两栏同一高度的文字检测成一行，所以识别前先在图片上找栏缝：在图片中间 30%–70% 宽度内，按 -4° 到 4°（步长 0.25°）逐条试斜线，统计穿过的深色像素。穿过的深色像素不到普通位置中位数的 10%、且缝宽不小于图片宽度 1.2% 时判为两栏。

判为两栏后，沿斜线把图片切成左右两张（另一半涂白）分别识别；右栏第一行没有缩进时接到左栏最后一段。检测到两栏图片时，页面自动切到两栏。

### 破折号修正

PaddleOCR 常把「——」识别成「一」。对每个「一」，按它在行内的位置估算图片坐标，在左右各 2.5 个字宽、行高 15%–80% 的范围内寻找最长的连续深色横线（每行用 Otsu 自适应阈值）。破折号约 2 个字宽，「一」不到 1 个字宽，超过 1.4 个字宽判为破折号。

浏览器控制台中的 `[Column]`、`[Dash]` 日志会输出每次判定的测量值，方便调整阈值。

## 已知限制

- 段落中局部加粗等字体样式无法还原
- 省略号「……」可能被识别为「..」
- 不保留原图的行内换行：表格、清单、诗歌等需要逐行排列的内容会被合并成段落
- 原图段落开头没有缩进、段间也没有空行时，相邻段落会被合并成一段
- 只支持单栏和两栏；栏缝过窄或有跨栏的图片、通栏标题时，可能识别为单栏
- 两栏时 A4 的分页位置与原图不一一对应；Word 中的分栏位置以 Word 排版为准
- 繁体中文识别效果一般
- 如果「一」旁边正好有很长的横线（表格线、下划线），可能被误判为破折号
- 首次使用需下载约 30MB 的模型和运行时

## 项目结构

```
index.html             页面、样式和全部逻辑
vendor/paddle-ocr.js   PaddleOCR 浏览器版（本地打包）
docs/screenshot.png    README 截图
```

## 重新打包 vendor/paddle-ocr.js

`@gutenye/ocr-browser` 通过 CDN 的 ESM 自动打包使用时，依赖会被打包坏，所以改为本地用 esbuild 打包成单个文件。

1. 安装依赖

```
npm i @gutenye/ocr-browser@1.4.9 onnxruntime-web esbuild
```

2. 新建入口文件 `entry.js`（`wasmPaths` 中的版本号与安装的 onnxruntime-web 版本保持一致）

```
import Ocr from '@gutenye/ocr-browser'
import * as ort from 'onnxruntime-web'
ort.env.wasm.wasmPaths = 'https://cdn.jsdelivr.net/npm/onnxruntime-web@1.30.0/dist/'
window.PaddleOcr = Ocr
```

3. 打包

```
npx esbuild entry.js --bundle --format=iife --minify --platform=browser --external:fs --external:path --external:crypto --outfile=vendor/paddle-ocr.js
```

识别模型在 `index.html` 的 `MODEL_BASE` 中配置，当前使用 `@gutenye/ocr-models@1.4.2` 中的 PP-OCRv4 检测 / 识别模型。

## 第三方依赖

| 依赖 | 用途 | 许可证 |
|---|---|---|
| [@gutenye/ocr](https://github.com/gutenye/ocr) | PaddleOCR 浏览器封装 | MIT |
| [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) PP-OCRv4 模型 | 文字检测与识别 | Apache-2.0 |
| [onnxruntime-web](https://github.com/microsoft/onnxruntime) | 模型推理 | MIT |
| [@techstark/opencv-js](https://github.com/TechStark/opencv-js) | 图像处理 | Apache-2.0 |
| [js-clipper](https://sourceforge.net/projects/jsclipper/) | 文本框多边形处理 | Boost |
| [docx](https://github.com/dolanmiu/docx) | 导出 Word | MIT |
| [html2canvas](https://github.com/niklasvh/html2canvas) | 导出 PDF（页面转图片） | MIT |
| [jsPDF](https://github.com/parallax/jsPDF) | 导出 PDF | MIT |

## 许可证

[MIT](LICENSE)
