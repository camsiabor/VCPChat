# 06 — 顶层功能模块 | Top-Level Feature Modules

## 概述 | Overview

VCPChat 将各独立功能封装为顶层目录模块。每个模块通常包含：

VCPChat organizes independent features as top-level directory modules. Each module typically contains:

- `*.html` — 子窗口的 HTML 结构（通过 `BrowserWindow.loadFile` 加载）
- `*.js` — 渲染进程逻辑（页面脚本）
- `*.css` — 模块专属样式
- `README.md` — 模块文档（部分模块有）

这些子窗口通过 `preload.js` 暴露的 `window.electronAPI` 与主进程通信。

These child windows communicate with the main process through `window.electronAPI` exposed by `preload.js`.

---

## 1. `Groupmodules/` — 群聊模块 | Group Chat Module

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `groupchat.js` | 群聊核心逻辑（主进程端，被 `main.js` 和 `groupChatHandlers.js` 引用）|
| `grouprenderer.js` | 群聊 UI 渲染（渲染进程端，渲染群组设置表单和群聊界面）|

### 架构 / Architecture

`groupchat.js` 运行在**主进程**，提供：

- 群组配置管理（CRUD）
- 群聊历史读写
- 多 Agent 轮流发言逻辑（sequential / naturerandom / inviteonly 三种模式）
- 群组 Canvas 集成（`{{VCPChatCanvas}}` 和 `{{VCPChatGroupSessionWatcher}}` 占位符）
- 话题总结触发

### 发言模式详解 / Speech Mode Details

```
sequential:
  成员按列表顺序轮流发言，每次由 invitePrompt 提示发言者

naturerandom:
  根据消息中的 @提及 和预设标签匹配，加权随机选择发言成员
  有保底发言者逻辑（无明确触发时随机选一个）

inviteonly:
  用户手动点击 Agent 头像旁的"邀请发言"按钮触发
```

### 关键函数 / Key Functions

```javascript
initializePaths(paths)                           // 初始化路径
getGroupSessionWatcher(groupId, topicId)         // 获取群会话监控信息
// (返回当前话题的消息统计、时间信息，供 AI 参考)
```

---

## 2. `Canvasmodules/` — 协同编辑器 | Canvas Collaborative Editor

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `canvas.html` | Canvas 窗口 HTML |
| `canvas.js` | Canvas 窗口逻辑 |
| `canvas.css` | Canvas 样式 |

### 核心功能 / Core Features

- **CodeMirror 5 编辑器** — 支持多语言语法高亮（JS、Python、HTML、Markdown 等）
- **双侧边栏布局** — 左侧文件列表，右侧版本历史
- **自动保存** — 2 秒防抖自动保存到 `AppData/canvas/`
- **版本历史** — 每次保存记录变更历史节点（时间轴可视化）
- **Diff 视图** — 检测外部变更时显示 diff，支持接受/拒绝
- **执行能力**:
  - `run-py-btn` — 运行 Python 代码（沙箱执行，通过 IPC 调用主进程）
  - `render-md-btn` — 渲染 Markdown 预览
  - `render-html-btn` — 渲染 HTML 预览
- **搜索过滤** — 实时过滤文件列表
- **右键菜单** — 重命名、复制、删除

### 与主聊天流的集成 / Integration with Chat Flow

当 AI 回复包含 `{{VCPChatCanvas}}` 占位符时，`messageRenderer.js` 将其渲染为"打开 Canvas"按钮，点击后打开 Canvas 窗口并加载对应文件。

### 数据存储 / Data Storage

```
AppData/canvas/
    {filename}                    ← 文件内容
    .history/
        {filename}/
            {timestamp}.json      ← 版本历史快照
```

---

## 3. `Musicmodules/` — 音乐播放器 | Music Player

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `music.html` | 音乐播放器窗口 HTML |
| `music.js` | 播放器逻辑 |
| `music.css` | 播放器样式 |
| `socket.io.min.js` | Socket.IO 客户端（本地副本）|

### 核心功能 / Core Features

- 本地音乐文件夹扫描（通过 `musicScannerWorker.js` Worker Thread）
- 音乐元数据读取（music-metadata 库）
- 专辑封面提取与缓存（`AppData/MusicCoverCache/`）
- 播放列表管理（`AppData/songlist.json`）
- 自定义播放列表
- 歌词显示（LRC 格式，通过 `lyricFetcher.js` 获取）
- 播放模式：顺序、随机、单曲循环
- **与 Rust 音频引擎通信**（所有音频处理委托给 `audio_engine/audio_server`）
- 均衡器（EQ）控制（FIR/IIR 两种模式）
- WASAPI 设备选择（Windows 独占模式）
- 升频设置（最高 192kHz）

---

## 4. `Assistantmodules/` — 迷你助手 | Mini-Assistant

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `assistant.html` | 主助手弹窗 HTML |
| `assistant.js` | 助手逻辑 |
| `assistant.css` | 助手样式 |
| `assistant-bar.html` | 助手工具条 HTML（悬浮在屏幕边缘）|
| `assistant-bar.js` | 助手工具条逻辑 |

### 核心功能 / Core Features

- 全局文本选中监听（通过 `selection-hook` 原生模块）
- 选中文本后弹出迷你助手窗口
- 快速 AI 操作（翻译、解释、总结、改写等）
- 可拖动的悬浮工具条
- 结果可直接发送到主聊天窗口

---

## 5. `Flowlockmodules/` — 心流锁 | Flow Lock

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `flowlock.js` | 心流锁前端逻辑 |
| `flowlock.css` | 发光效果样式 |
| `flowlock-integration.js` | 与 VCPDistributedServer 插件的集成桥接 |
| `README.md` | 详细文档 |

### 核心功能 / Core Features

- **自动续写循环**: AI 每次回复结束后自动触发下一次
- **发光 UI 效果**: 心流锁激活时标题栏持续脉冲发光
- **智能重试**: 失败时自动重试，最多 3 次
- **双向控制**:
  - 用户：右键菜单、中键点击、`Ctrl/Cmd+G` 快捷键
  - AI：通过 VCP 工具调用 `Flowlock.start/stop/promptee/prompter`
- **续写提示词**: 可配置触发下次 AI 发言的提示词

---

## 6. `Forummodules/` — 论坛 | Forum

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `forum.html` | 论坛窗口 HTML |
| `forum.js` | 论坛逻辑 |
| `forum.css` | 论坛样式 |
| `README.md` | 功能文档 |

### 核心功能 / Core Features

- 连接 VCPToolBox 后端的论坛 API（HTTP Basic Auth）
- 浏览帖子列表（支持置顶、板块筛选、搜索）
- 查看帖子详情（Markdown 渲染）
- 发表回复、发布新帖
- 删除帖子/楼层
- "记住凭据"功能（配置存储于 `AppData/forum.config.json`）

### API 端点 / API Endpoints

```
GET  /admin_api/forum/posts           帖子列表
GET  /admin_api/forum/post/:uid       帖子详情
POST /admin_api/forum/reply/:uid      发表回复
POST /admin_api/forum/new             发布新帖
DELETE /admin_api/forum/post/:uid     删除帖子/楼层
```

---

## 7. `Promptmodules/` — 系统提示词管理 | System Prompt Management

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `prompt-manager.js` | 三种模式的统一管理器 |
| `original-prompt-module.js` | 原始富文本模式 |
| `modular-prompt-module.js` | 模块化积木块模式 |
| `preset-prompt-module.js` | 临时与预制模式 |
| `prompt-modules.css` | 样式 |
| `README.md` | 详细文档 |

### 三种提示词模式 / Three Prompt Modes

| 模式 | 存储字段 | 特点 |
|------|---------|------|
| **原始富文本** | `originalSystemPrompt` | 传统文本域，直接编辑 |
| **模块化积木块** | `advancedSystemPrompt` | 可拖拽排序的多条目模块，支持"小仓"片段管理 |
| **临时与预制** | `presetSystemPrompt` | 从文件夹加载预设，支持占位符替换 |

---

## 8. `Voicechatmodules/` — 语音聊天 | Voice Chat

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `voicechat.html` | 语音聊天窗口 HTML |
| `voicechat.js` | 语音聊天逻辑 |
| `voicechat.css` | 样式 |
| `recognizer.html` | 语音识别子页面 |

### 核心功能 / Core Features

- 麦克风录音 + 语音识别（Web Speech API / 系统 ASR）
- 识别结果自动填入输入框
- TTS 播放（SoVITS 或系统 TTS）
- 实时波形可视化

---

## 9. `RAGmodules/` — RAG 观察器 | RAG Observer

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `RAG_Observer.html` | 灵视中心观察器窗口 HTML |
| `rag-observer-config.js` | 配置和 WebSocket 连接逻辑 |

### 核心功能 / Core Features

- 连接 VCP 后端 WebSocket（`ws://vcpserver/vcpinfo/VCP_Key=...`）
- 实时可视化显示 RAG 检索过程（记忆检索、Agent 链、Memo 等）
- 用于调试和理解 AI 的记忆调用行为
- 毛玻璃风格 UI，多颜色标识不同类型事件

---

## 10. `Themesmodules/` — 主题管理 | Theme Management

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `themes.html` | 主题选择器窗口 HTML |
| `themes.js` | 主题切换逻辑 |
| `themes-module.css` | 主题模块样式 |

### 核心功能 / Core Features

- 内置主题列表（暗色、亮色、樱花夜等）
- 壁纸选择（从本地目录加载，含缩略图预览）
- 自定义 CSS 主题注入
- 与全局设置中的主题配置联动

---

## 11. `Translatormodules/` — 翻译器 | Translator

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `translator.html` | 翻译器窗口 HTML |
| `translator.js` | 翻译逻辑 |
| `translator.css` | 样式 |

### 核心功能 / Core Features

- 调用 VCP 后端 AI 进行文本翻译
- 支持多语言切换
- 常驻浮动窗口（单例模式）
- 与助手模块联动（选中文本→快速翻译）

---

## 12. `Notemodules/` — 笔记 | Notes

### 核心功能 / Core Features

- 浏览和编辑 VCPToolBox 后端的网络笔记（类似文件树结构）
- 笔记内容 Markdown 渲染
- 本地缓存（`AppData/network-notes-cache.json`）
- 支持笔记附件存储

---

## 13. `Memomodules/` — 备忘录 | Memo

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `memo.html` | 备忘录窗口 HTML |
| `memo.js` | 备忘录逻辑 |
| `memo.css` | 样式 |

### 核心功能 / Core Features

- 快速记录卡片式备忘
- 支持 Markdown 格式
- 本地持久化存储

---

## 14. `Dicemodules/` — 骰子 | Dice

### 文件 / Files

| 文件 | 说明 |
|------|------|
| `dice.html` | 骰子窗口 HTML |
| `dice.js` | 骰子逻辑 |
| `dice.css` | 样式 |

### 核心功能 / Core Features

- 3D 骰子模拟（@3d-dice/dice-box 库）
- AI 可通过 VCP 分布式服务器插件触发骰子投掷
- 投掷结果广播到聊天界面

---

*→ 下一章: [07_audio_engine.md](./07_audio_engine.md)*  
*← 上一章: [05_modules_renderer.md](./05_modules_renderer.md)*
