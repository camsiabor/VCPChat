# 01 — 整体架构概览 | Overall Architecture Overview

## 1. 项目定位 | Project Purpose

VCPChat 是一个基于 **Electron** 的桌面端 AI 聊天客户端，专为 **VCP（Variable & Command Protocol）** 后端服务器（VCPToolBox）设计。  
VCPChat is an **Electron**-based desktop AI chat client specifically designed for the **VCP (Variable & Command Protocol)** backend server (VCPToolBox).

- 前端负责：渲染界面、管理 Agent、展示消息流、播放音频、协同编辑等。  
  Frontend responsibilities: UI rendering, Agent management, message stream display, audio playback, collaborative editing, etc.
- 后端（VCPToolBox）负责：AI 推理代理、工具调用、记忆库管理、分布式节点协调。  
  Backend (VCPToolBox) responsibilities: AI inference proxy, tool invocation, memory management, distributed node coordination.

---

## 2. 技术栈 | Technology Stack

| 层 Layer | 技术 Technology |
|----------|----------------|
| 桌面框架 Desktop Framework | [Electron](https://www.electronjs.org/) v37+ |
| 前端语言 Frontend Language | Vanilla JavaScript (ES6+), HTML5, CSS3 |
| 主进程 Main Process | Node.js (CommonJS modules) |
| 构建工具 Build Tool | electron-builder |
| 原生模块 Native Modules | node-gyp, @electron/rebuild, selection-hook |
| 音频引擎 Audio Engine | Rust (audio_server binary) + Python (重采样扩展 / resampler extension) |
| 协同编辑 Collaborative Editor | CodeMirror 5 |
| 数学渲染 Math Rendering | KaTeX |
| Markdown 渲染 Markdown Rendering | marked.js + DOMPurify |
| 代码高亮 Syntax Highlighting | highlight.js |
| 图表渲染 Diagram Rendering | Mermaid.js |
| 3D 渲染 3D Rendering | Three.js |
| 动画 Animations | Anime.js |
| 向量搜索 Vector Search | hnswlib-node |
| 全文搜索 Full-text Search | FlexSearch |
| 文件解析 File Parsing | mammoth (docx), pdf-parse, tesseract.js (OCR) |
| HTTP 客户端 HTTP Client | node-fetch, axios |
| WebSocket | ws |
| 文件监听 File Watch | chokidar |
| 定时任务 Scheduling | node-schedule |

---

## 3. 三层进程架构 | Three-Layer Process Architecture

Electron 应用由三层构成：

```
┌─────────────────────────────────────────────────────┐
│                   Main Process                      │
│  main.js  — Node.js 环境，完整 OS 访问权限           │
│  • 窗口管理 Window management                        │
│  • IPC 路由 IPC routing (ipcMain)                   │
│  • 文件 I/O File I/O                                │
│  • 音频引擎进程管理 Audio engine process mgmt        │
│  • 分布式服务器 Distributed server                   │
│  • 系统托盘 System tray                             │
└─────────────────┬───────────────────────────────────┘
                  │  contextBridge (preload.js)
                  │  安全 API 暴露 / Secure API exposure
┌─────────────────▼───────────────────────────────────┐
│                 Renderer Process                    │
│  renderer.js + main.html — 受沙箱约束               │
│  • 聊天 UI Chat UI                                  │
│  • Agent/Group 管理 Management                      │
│  • 消息渲染 Message rendering                       │
│  • 话题管理 Topic management                        │
│  • 主题/样式 Theming/styling                        │
│  • 模块编排 Module orchestration                    │
└─────────────────────────────────────────────────────┘

外部子窗口 / External Child Windows (BrowserWindow):
  • MusicWindow (Musicmodules/)
  • CanvasWindow (Canvasmodules/)
  • ForumWindow (Forummodules/)
  • AssistantBar (Assistantmodules/)
  • TranslatorWindow (Translatormodules/)
  • RAGObserverWindow (RAGmodules/)
  • NotesWindow (Notemodules/)
  • VoiceChatWindow (Voicechatmodules/)
  • DiceWindow (Dicemodules/)
  • MemoWindow (Memomodules/)
  • ImageViewerWindow (modules/image-viewer.html)
  • TextViewerWindow (modules/text-viewer.html)
```

---

## 4. 核心数据流 | Core Data Flow

### 4.1 发送消息流程 | Sending a Message

```
用户输入 User Input
  │
  ▼
renderer.js (chatManager.sendMessage)
  │  收集附件、正则预处理 / collect attachments, regex pre-process
  ▼
preload.js (contextBridge.electronAPI)
  │  IPC invoke: 'send-chat-message'
  ▼
main.js → modules/ipc/chatHandlers.js
  │  构建消息数组、注入上下文 / build message array, inject context
  ▼
modules/vcpClient.js (sendToVCP)
  │  HTTP POST → VCP 服务器 / VCP server
  ▼
VCP 后端 / VCP Backend (VCPToolBox)
  │  AI 推理、工具调用、记忆检索
  ▼
流式响应 Streaming Response (SSE/chunks)
  │
  ▼
main.js → webContents.send('vcp-stream-event', chunk)
  │
  ▼
renderer.js (chatManager 监听)
  │
  ▼
modules/messageRenderer.js
  │  渲染气泡、Markdown、工具调用块、媒体等
  ▼
chatMessages DOM
```

### 4.2 工具调用流程 | Tool Call Flow

```
AI 响应包含工具调用标记:
<<<[TOOL_REQUEST]>>>
tool_name:「始」ToolName「末」,
param:「始」value「末」
<<<[END_TOOL_REQUEST]>>>
          │
          ▼
VCP 后端解析并执行工具
          │
          ▼ (异步结果通过 WebSocket 推送)
VcpLog WebSocket (ws://vcpserver:port/vcpinfo)
          │
          ▼
main.js WebSocket 监听器 → 主窗口渲染器通知
          │
          ▼
notificationRenderer.js → 通知侧边栏
```

---

## 5. 目录结构速览 | Directory Structure at a Glance

```
VCPChat/
├── main.js                    # Electron 主进程入口 / Main process entry
├── preload.js                 # 上下文桥接 / Context bridge
├── renderer.js                # 渲染进程主逻辑 / Renderer main logic
├── main.html                  # 主窗口 HTML / Main window HTML
├── style.css                  # 全局样式（遗留）/ Global styles (legacy)
├── themes.css                 # 主题（遗留）/ Themes (legacy)
├── styles/                    # 模块化 CSS / Modular CSS
│   ├── base.css               # 基础重置与变量 / Base reset & variables
│   ├── layout.css             # 布局 / Layout
│   ├── chat.css               # 聊天区域 / Chat area
│   ├── components.css         # 通用组件 / Common components
│   ├── settings.css           # 设置面板 / Settings panel
│   ├── search.css             # 搜索 / Search
│   ├── notifications.css      # 通知 / Notifications
│   ├── animations.css         # 动画 / Animations
│   ├── messageRenderer.css    # 消息渲染专属 / Message renderer specific
│   └── themes.css             # 主题变量 / Theme variables
├── modules/                   # 主进程 + 渲染进程共享模块 / Shared modules
│   ├── ipc/                   # IPC 处理器（主进程端）/ IPC handlers (main process)
│   ├── renderer/              # 渲染子模块（渲染进程）/ Renderer sub-modules
│   ├── utils/                 # 工具类 / Utilities
│   ├── chatManager.js         # 聊天核心逻辑 / Core chat logic
│   ├── vcpClient.js           # VCP HTTP 客户端 / VCP HTTP client
│   ├── messageRenderer.js     # 消息渲染主模块 / Message rendering main module
│   └── ...                    # 详见 03_modules_core.md
├── Assistantmodules/          # 迷你助手侧边栏 / Mini-assistant sidebar
├── Canvasmodules/             # 协同编辑器 / Collaborative editor
├── Dicemodules/               # 骰子模块 / Dice module
├── Flowlockmodules/           # 心流锁 / Flow lock
├── Forummodules/              # 论坛界面 / Forum interface
├── Groupmodules/              # 群聊逻辑 / Group chat logic
├── Memomodules/               # 备忘录 / Memo
├── Musicmodules/              # 音乐播放器 / Music player
├── Notemodules/               # 笔记 / Notes
├── Promptmodules/             # 系统提示词管理 / System prompt management
├── RAGmodules/                # RAG 观察器 / RAG observer
├── Themesmodules/             # 主题管理 / Theme management
├── Translatormodules/         # 翻译器 / Translator
├── Voicechatmodules/          # 语音聊天 / Voice chat
├── audio_engine/              # 预编译 Rust 音频服务器 / Pre-built Rust audio server
├── rust_audio_engine/         # Rust 音频重采样源码 / Rust audio resampler source
├── VCPDistributedServer/      # 内嵌分布式服务器 / Embedded distributed server
├── VchatManager/              # VChat 管理工具（独立 Electron 应用）/ VChat manager app
├── VCPHumanToolBox/           # 人类工具调用器（独立 Electron 应用）/ Human tool invoker app
├── SovitsTest/                # SoVITS TTS 测试脚本 / SoVITS TTS test scripts
├── migration/                 # 数据迁移脚本 / Data migration scripts
├── vendor/                    # 第三方库本地副本 / Vendored third-party libraries
├── assets/                    # 静态资产（图标、图片）/ Static assets (icons, images)
├── AppData/                   # 运行时数据（不入 Git）/ Runtime data (not in Git)
│   ├── Agents/                # Agent 配置 / Agent configs
│   ├── AgentGroups/           # 群组配置 / Group configs
│   ├── UserData/              # 聊天历史、附件 / Chat history, attachments
│   ├── settings.json          # 全局设置 / Global settings
│   └── ...
└── 0_summary/                 # 本文档目录 / This documentation directory
```

---

## 6. 关键设计决策 | Key Design Decisions

| 决策 Decision | 说明 Rationale |
|--------------|---------------|
| 数据存储在项目目录内 Data stored inside project dir | `AppData/` 与代码同目录，便于备份和迁移，无需关心 OS 用户目录。/ Keeps data co-located with code for easy backup and migration without OS user-dir concerns. |
| 懒加载重型依赖 Lazy-load heavy deps | `chokidar`、`speechRecognizer` 等在需要时才 `require`，加快启动速度。/ `chokidar`, `speechRecognizer`, etc. are `require()`d on demand to speed up startup. |
| IPC 模块化 Modular IPC | 每个功能域有独立的 IPC handler 文件（`modules/ipc/`），主进程 `main.js` 只做路由注册。/ Each domain has its own IPC handler file; `main.js` only registers them. |
| Vendor 本地化 Vendored libraries | 核心前端库（marked, highlight, KaTeX 等）存储在 `vendor/`，无需网络访问，保证离线可用。/ Core frontend libs stored in `vendor/` for offline availability. |
| Rust 音频引擎 Rust audio engine | Python 音频处理性能不足，迁移到 Rust 原生二进制（`audio_server`），通过本地 HTTP 与 Electron 通信。/ Migrated from Python to Rust native binary for better audio performance; communicates over local HTTP. |
| VCP 协议文本标记 Text-marker VCP protocol | 工具调用使用 `<<<[TOOL_REQUEST]>>>` 文本标记而非 JSON Function Calling，兼容所有 LLM，无需特定 API 字段支持。/ Text-marker-based tool calls compatible with all LLMs, no specific API field required. |

---

## 7. 外部依赖关系 | External Dependencies

```
VCPChat (前端 Frontend)
    │
    ├──HTTP(S) POST /v1/chat/completions──► VCPToolBox (后端 Backend)
    │                                          │
    │                                          ├── AI Model API (OpenAI-compatible)
    │                                          ├── Plugin System (VCP tools)
    │                                          └── Memory / RAG System
    │
    ├──WebSocket ws://vcpserver/vcpinfo──────► VCPToolBox (实时通知 Real-time notifications)
    │
    ├──HTTP localhost:63789──────────────────► Rust Audio Server (audio_engine/)
    │
    ├──WebSocket VCPDistributedServer──────── VCPToolBox (分布式节点 Distributed node)
    │
    └──SoVITS API (可选 optional)──────────► SoVITS TTS Server
```

---

*→ 下一章: [02_electron_core.md](./02_electron_core.md)*
