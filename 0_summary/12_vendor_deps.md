# 12 — 第三方库与依赖 | Third-Party Libraries & Dependencies

## 概述 | Overview

VCPChat 的依赖分为两类：

1. **npm 包依赖** — 在 `package.json` 中声明，通过 `npm install` 安装到 `node_modules/`
2. **Vendor 本地化库** — 存储在 `vendor/` 目录，直接在 HTML 中通过 `<script>` 标签引用（确保离线可用）

VCPChat dependencies fall into two categories:

1. **npm packages** — declared in `package.json`, installed via `npm install` to `node_modules/`
2. **Vendored libraries** — stored in `vendor/`, referenced directly in HTML via `<script>` tags (ensuring offline availability)

---

## 1. `vendor/` — 本地化前端库 | Vendored Frontend Libraries

```
vendor/
├── marked.min.js            # Markdown 解析器
├── purify.min.js            # DOMPurify — HTML 净化/XSS 防护
├── highlight.min.js         # 代码语法高亮
├── atom-one-dark.min.css    # highlight.js 深色主题
├── atom-one-light.min.css   # highlight.js 亮色主题
├── katex.min.js             # 数学公式渲染
├── katex.min.css            # KaTeX 样式
├── auto-render.min.js       # KaTeX 自动渲染（扫描 DOM）
├── mermaid.min.js           # 图表/流程图渲染
├── three.min.js             # Three.js 3D 渲染
├── three.module.js          # Three.js ES Module 版本
├── anime.min.js             # Anime.js 动画库
├── morphdom.min.js          # DOM diffing（流式更新优化）
├── html2canvas.min.js       # DOM → Canvas 截图
├── Sortable.min.js          # 拖拽排序
├── pyodide.js               # Pyodide — 浏览器中运行 Python
└── pyodide.asm.js           # Pyodide WebAssembly
```

### 各库用途 | Library Usage

| 库 Library | 版本特征 | 使用模块 |
|-----------|---------|---------|
| **marked** | 轻量 MD 解析 | `messageRenderer.js`, `contentProcessor.js` |
| **DOMPurify** | XSS 净化 | 所有 Markdown 输出前净化 |
| **highlight.js** | 代码高亮 | 消息中代码块 |
| **KaTeX** | 数学公式 | 消息中 `$...$` 和 `$$...$$` |
| **Mermaid** | 流程图/时序图 | 消息中 mermaid 代码块 |
| **Three.js** | 3D 渲染 | 骰子动画、部分 AI 生成特效 |
| **Anime.js** | 动画编排 | `animation.js` 中的动效 |
| **morphdom** | 最小 DOM 变更 | `streamManager.js` 流式更新 |
| **html2canvas** | 截图 | 消息导出为图片 |
| **Sortable.js** | 拖拽排序 | Agent 列表、话题列表、提示词模块 |
| **Pyodide** | WebAssembly Python | Canvas 中运行 Python 代码 |

---

## 2. npm 依赖分类 | npm Dependencies by Category

### 2.1 Electron 框架 | Electron Framework

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `electron` | ^37.2.6 | 桌面应用框架 |
| `@electron/rebuild` | ^3.7.2 | 原生模块重编译 |
| `electron-builder` | ^26.0.12 | 打包/发布 |

### 2.2 原生模块 | Native Modules

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `selection-hook` | ^0.9.23 | 全局文本选中监听（系统级钩子）|
| `node-pty` | ^1.1.0-beta37 | 伪终端（终端模拟器）|
| `node-addon-api` | ^8.3.1 | 原生 addon 开发支持 |
| `node-gyp` | ^11.4.2 | 原生模块编译工具 |
| `node-global-key-listener` | ^0.3.0 | 全局键盘监听 |
| `clipboard-event` | ^1.6.0 | 剪贴板事件监听 |

### 2.3 文件操作 | File Operations

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `fs-extra` | ^11.3.2 | 增强版 fs（ensureDir, copy, move 等）|
| `chokidar` | ^4.0.3 | 文件系统监听（懒加载）|
| `glob` | ^10.0.0 | 文件路径模式匹配 |
| `trash` | ^9.0.0 | 安全移入垃圾桶（而非永久删除）|
| `tmp` | ^0.2.5 | 临时文件管理 |

### 2.4 媒体处理 | Media Processing

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `music-metadata` | ^11.4.0 | 音频文件元数据提取（在 Worker Thread 中）|
| `sharp` | ^0.34.2 | 图片处理（壁纸缩略图生成、图片压缩）|
| `tesseract.js` | ^6.0.1 | OCR 图片文字识别 |

### 2.5 文档解析 | Document Parsing

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `mammoth` | ^1.9.1 | Word 文档（.docx）转 HTML/文本 |
| `pdf-parse` | ^1.1.1 | PDF 文本提取 |
| `pdf-poppler` | ^0.2.1 | PDF 转图片（poppler 工具封装）|
| `exceljs` | ^4.4.0 | Excel 文件读写 |
| `iconv-lite` | ^0.6.3 | 字符编码转换（GBK、Big5 等）|

### 2.6 网络 | Networking

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `axios` | ^1.10.0 | HTTP 客户端 |
| `node-fetch` | ^3.3.2 | Fetch API (Node.js 端) |
| `ws` | ^8.17.0 | WebSocket 服务器/客户端 |
| `express` | ^5.1.0 | HTTP 服务器（分布式服务器用）|
| `portfinder` | ^1.0.37 | 自动查找可用端口 |
| `puppeteer` | ^24.15.0 | Headless Chrome（网页内容抓取）|
| `cheerio` | ^1.1.2 | HTML 解析（类 jQuery，服务端）|
| `turndown` | ^7.2.1 | HTML → Markdown 转换 |

### 2.7 搜索与数据 | Search & Data

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `flexsearch` | ^0.8.212 | 全文搜索索引 |
| `clientside-search` | ^1.8.1 | 客户端搜索（备用）|
| `hnswlib-node` | ^3.0.0 | HNSW 向量数据库（本地 RAG）|
| `diff-match-patch` | ^1.0.5 | 文本 diff 计算 |

### 2.8 工具 | Utilities

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `dotenv` | ^17.2.3 | 环境变量加载（`.env` 文件）|
| `node-schedule` | ^2.1.1 | 定时任务 |
| `os` | ^0.1.2 | OS 信息（标准模块包装）|
| `crypto` | (内置/built-in) | SHA-256 哈希（文件去重）|
| `minimatch` | ^9.0.0 | Glob 模式匹配 |

### 2.9 UI 增强 | UI Enhancement

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `codemirror` | ^5.65.19 | Canvas 编辑器（CodeMirror 5）|
| `xterm` | ^5.3.0 | 终端模拟器（xterm.js）|
| `xterm-addon-fit` | ^0.8.0 | xterm 适配容器大小 |
| `three` | ^0.178.0 | Three.js（npm 版本）|
| `animejs` | ^4.0.2 | Anime.js（npm 版本）|
| `morphdom` | ^2.7.7 | DOM diffing |
| `html2canvas` | ^1.4.1 | DOM 截图 |

### 2.10 代码质量 | Code Quality

| 包 Package | 版本 Version | 用途 |
|-----------|------------|------|
| `eslint` | ^9.39.1 | JavaScript 代码检查 |
| `stylelint` | ^16.25.0 | CSS 代码检查 |

---

## 3. Python 依赖 | Python Dependencies

`requirements.txt` 中的依赖（用于可选功能）：

```
# 主要用于音频处理和 SoVITS TTS 相关功能
# 具体内容见 requirements.txt
```

`pyproject.toml` / `poetry.lock` 定义了 Python 项目结构（可选的 Poetry 依赖管理）。

---

## 4. 依赖安全注意事项 | Dependency Security Notes

| 注意点 | 说明 |
|-------|------|
| `selection-hook` | 系统级钩子，需要原生编译，安装后应通过 `electron-rebuild` 重编译 |
| `puppeteer` | 内含 Chromium 下载，体积较大（约 300MB），用于网页抓取 |
| `hnswlib-node` | 原生 C++ 模块，需要编译环境 |
| `node-pty` | 原生模块，依赖系统终端接口 |
| `sharp` | 原生模块，体积较大，需要系统图像库 |
| `DOMPurify` (vendor) | 所有 AI 生成的 HTML 内容必须经过 DOMPurify 净化，防止 XSS |

---

## 5. `.npmrc` 配置

```
# 通常包含 registry 配置或代理设置
# 具体内容见 .npmrc 文件
```

---

*← 上一章: [11_companion_tools.md](./11_companion_tools.md)*  
*↑ 返回索引: [README.md](./README.md)*
