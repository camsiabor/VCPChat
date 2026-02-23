# 02 — Electron 核心层 | Electron Core Layer

## 概述 | Overview

VCPChat 的 Electron 核心由三个文件构成，对应 Electron 的标准三层架构：

VCPChat's Electron core consists of three files corresponding to Electron's standard tri-layer architecture:

| 文件 File | 进程 Process | 职责 Responsibility |
|-----------|-------------|---------------------|
| `main.js` | Main Process | 系统级操作、窗口管理、IPC 路由、后台服务 |
| `preload.js` | Preload Script | 安全地将 Main 能力暴露给 Renderer |
| `renderer.js` | Renderer Process | UI 逻辑、聊天管理、状态管理 |
| `main.html` | Renderer Process | 主窗口 HTML 结构 |

---

## 1. `main.js` — 主进程 | Main Process

### 1.1 职责概要 | Responsibility Summary

`main.js` 是 Electron 应用的入口，运行在 Node.js 完整环境中（无浏览器沙箱限制）。

`main.js` is the Electron application entry point, running in a full Node.js environment (no browser sandbox restrictions).

### 1.2 主要功能模块 | Major Functional Blocks

#### 启动流程 | Startup Sequence

```
app.whenReady()
  ├── 初始化路径常量 (APP_DATA_ROOT_IN_PROJECT, AGENT_DIR, USER_DATA_DIR, ...)
  ├── 创建必要目录 (fs.ensureDirSync)
  ├── 注册所有 IPC Handler (各 modules/ipc/*.js)
  ├── 创建主窗口 (BrowserWindow → main.html)
  ├── 启动 Rust 音频引擎 (startAudioEngine → audio_engine/audio_server)
  ├── 加载设置并连接 VCP Log WebSocket
  └── 初始化分布式服务器 (VCPDistributedServer)
```

#### 窗口管理 | Window Management

主进程管理以下窗口：

- **mainWindow** — 主聊天窗口
- **translatorWindow** — 翻译器（单例）
- **ragObserverWindow** — RAG 观察器（单例）
- **openChildWindows[]** — 其他子窗口（音乐、Canvas、论坛、笔记、语音等）
- **tray** — 系统托盘图标

#### 路径常量 | Path Constants

```javascript
PROJECT_ROOT                         // __dirname (main.js 所在目录)
APP_DATA_ROOT_IN_PROJECT             // PROJECT_ROOT/AppData
AGENT_DIR                            // AppData/Agents
USER_DATA_DIR                        // AppData/UserData
SETTINGS_FILE                        // AppData/settings.json
MUSIC_PLAYLIST_FILE                  // AppData/songlist.json
MUSIC_COVER_CACHE_DIR                // AppData/MusicCoverCache
NETWORK_NOTES_CACHE_FILE             // AppData/network-notes-cache.json
WALLPAPER_THUMBNAIL_CACHE_DIR        // AppData/WallpaperThumbnailCache
RESAMPLE_CACHE_DIR                   // AppData/ResampleCache
CANVAS_CACHE_DIR                     // AppData/canvas
NOTES_MODULE_DIR                     // AppData/Notemodules
```

#### 文件监听器 | File Watcher (`fileWatcher` 对象)

用于监听聊天历史文件的外部修改（如用户直接编辑 JSON 文件）。  
Used to watch chat history files for external modifications (e.g., user directly editing JSON files).

```javascript
fileWatcher.watchFile(filePath, callback)    // 开始监听
fileWatcher.stopWatching()                   // 停止监听
fileWatcher.signalInternalSave()             // 标记内部保存，避免误触发
fileWatcher.setEditingMode(editing)          // 设置编辑状态
```

**防抖机制 / Debounce Logic**: 使用时间窗口（`INTERNAL_SAVE_WINDOW_MS = 2000ms`）区分内部保存触发和外部修改触发。

#### 音频引擎管理 | Audio Engine Management

```javascript
startAudioEngine()    // 启动 audio_engine/audio_server[.exe]
stopAudioEngine()     // 终止音频引擎进程
```

- 启动时等待 `RUST_AUDIO_ENGINE_READY` 输出信号（10 秒超时）。
- 通信端口：`63789`（通过本地 HTTP REST API）。

#### VCP Log WebSocket 连接 | VCP Log WebSocket Connection

主进程维持一个到 VCP 后端的 WebSocket 连接（`ws://vcpserver/vcpinfo/VCP_Key=...`），接收实时日志和异步工具调用结果，通过 `webContents.send` 转发给渲染进程。

The main process maintains a WebSocket connection to the VCP backend to receive real-time logs and async tool call results, forwarding them to the renderer via `webContents.send`.

#### IPC Handler 注册 | IPC Handler Registration

主进程从 `modules/ipc/` 导入所有处理器模块并逐一调用 `initialize(mainWindow, context)` 注册：

```javascript
windowHandlers.initialize(...)
settingsHandlers.initialize(...)
fileDialogHandlers.initialize(...)
agentHandlers.initialize(...)
regexHandlers.initialize(...)
chatHandlers.initialize(...)
groupChatHandlers.initialize(...)
sovitsHandlers.initialize(...)
promptHandlers.initialize(...)
notesHandlers.initialize(...)
assistantHandlers.initialize(...)
musicHandlers.initialize(...)
diceHandlers.initialize(...)
themeHandlers.initialize(...)
emoticonHandlers.initialize(...)
forumHandlers.initialize(...)
memoHandlers.initialize(...)
canvasHandlers.initialize(...)
```

---

## 2. `preload.js` — 预加载脚本（上下文桥）| Preload Script (Context Bridge)

### 2.1 作用 | Purpose

`preload.js` 在 Renderer 进程的沙箱环境中运行，是 Main Process 与 Renderer Process 之间的**安全桥梁**。它通过 `contextBridge.exposeInMainWorld` 将白名单 IPC 通道暴露为 `window.electronAPI` 和 `window.electron` 对象。

`preload.js` runs in the Renderer's sandbox and acts as the **secure bridge** between Main and Renderer processes. It exposes whitelisted IPC channels as `window.electronAPI` and `window.electron` objects via `contextBridge.exposeInMainWorld`.

### 2.2 暴露的 API 命名空间 | Exposed API Namespaces

#### `window.electron` (音乐播放器专用 / Music player specific)

```javascript
electron.send(channel, data)      // 单向发送（白名单频道）
electron.invoke(channel, data)    // 请求-响应
electron.on(channel, func)        // 监听主进程推送
```

频道白名单包括：`music-load`, `music-play`, `music-pause`, `music-seek`, `music-get-state`, `music-set-volume`, `music-get-devices`, `music-configure-output`, `music-set-eq`, 等。

#### `window.electronAPI` (通用 API / General API)

涵盖所有主要功能域：

```
Settings:       loadSettings, saveSettings, saveUserAvatar, saveAvatarColor
Agents:         getAgents, getAgentConfig, saveAgentConfig, createAgent, deleteAgent,
                getCachedModels, refreshModels, getHotModels, getFavoriteModels,
                toggleFavoriteModel, getAllItems, importRegexRules, updateAgentConfig,
                getGlobalWarehouse, saveGlobalWarehouse
Prompt Modules: loadPresetPrompts, loadPresetContent, selectDirectory,
                getActiveSystemPrompt, programmaticSetPromptMode
Topics:         getAgentTopics, createNewTopicForAgent, saveAgentTopicTitle,
                deleteTopic, getUnreadTopicCounts, updateTopicLock,
                exportTopicToMarkdown, exportTopicToHtml, saveTopicOrder
Chat:           getChatHistory, saveChatHistory, deleteMessage,
                searchMessages, getAgentChatStats
Groups:         getAgentGroups, createNewGroup, deleteGroup, saveGroupConfig,
                getGroupChatHistory, saveGroupChatHistory, getGroupTopics,
                createGroupTopic, saveGroupTopicTitle, deleteGroupTopic,
                saveGroupTopicOrder
File & Attach:  selectFile, saveAttachment, getAttachmentUrl,
                openFileInViewer, openImageInViewer, openTextInViewer
Canvas:         getCanvasFiles, openCanvasWindow, saveCanvasFile,
                deleteCanvasFile, renameCanvasFile, runCanvasPython,
                getCanvasFileHistory
Forum:          openForumWindow, getForumConfig, saveForumConfig
Music:          openMusicWindow, getMusicPlaylist, saveMusicPlaylist, ...
Notes:          openNotesWindow, getNetworkNotes, saveNetworkNotesCache, ...
Memo:           openMemoWindow, getMemos, saveMemo, deleteMemo
Dice:           openDiceWindow
TTS (SoVITS):   getSovitsModels, testSovitsTts, generateTts, ...
Speech:         startSpeechRecognition, stopSpeechRecognition
Regex:          getAgentRegex, saveAgentRegex
Theme:          getThemes, setTheme, getCustomTheme, setCustomTheme
Window:         minimizeWindow, maximizeWindow, closeWindow, ...
Assistant:      openAssistantBar, ...
RAG:            openRagObserver
```

#### `window.electronPath`

```javascript
electronPath.dirname(p)    // path.dirname
electronPath.extname(p)    // path.extname
electronPath.basename(p)   // path.basename
```

---

## 3. `renderer.js` — 渲染进程主逻辑 | Renderer Main Logic

### 3.1 职责 | Responsibilities

`renderer.js` 是渲染进程的根脚本，负责：

1. **全局状态管理** — 维护 `currentSelectedItem`, `currentTopicId`, `currentChatHistory`, `globalSettings`, `attachedFiles` 等核心状态。
2. **模块协调** — 初始化并串联所有渲染侧模块（chatManager, settingsManager, itemListManager 等）。
3. **事件路由** — 监听 DOM 事件、IPC 推送事件，分发到对应模块。
4. **UI 全局操作** — 主题切换、侧边栏宽度调整、通知面板开关、标签页切换。

`renderer.js` is the root renderer script responsible for global state management, module coordination, event routing, and global UI operations.

### 3.2 全局状态结构 | Global State Structure

```javascript
// 当前选中的 Agent 或群组
currentSelectedItem = {
    id: null,       // agentId 或 groupId
    type: null,     // 'agent' | 'group'
    name: null,
    avatarUrl: null,
    config: null    // 完整配置对象
}

// 当前话题 ID
currentTopicId = null

// 当前聊天历史（消息数组）
currentChatHistory = []

// 全局设置
globalSettings = {
    sidebarWidth, enableMiddleClickQuickAction,
    middleClickQuickAction, userName, filterEnabled,
    filterRules, enableRegenerateConfirmation,
    flowlockContinueDelay, enableThoughtChainInjection,
    ...
}

// 附件文件列表
attachedFiles = []

// TTS 相关状态
ttsAudioQueue, isTtsPlaying, currentPlayingMsgId, currentTtsSessionId
```

### 3.3 与其他模块的关系 | Relationship with Other Modules

```
renderer.js
  ├── 初始化 chatManager.init(...)
  ├── 初始化 settingsManager.init(...)
  ├── 初始化 itemListManager.init(...)
  ├── 初始化 topicListManager.init(...)
  ├── 初始化 messageRenderer (通过 import)
  ├── 初始化 notificationRenderer (通过 import)
  ├── 初始化 groupRenderer (Groupmodules/grouprenderer.js)
  ├── 初始化 searchManager
  ├── 初始化 filterManager
  ├── 初始化 flowlockModule (Flowlockmodules/)
  └── 初始化 promptManager (Promptmodules/)
```

### 3.4 `main.html` — 主窗口结构 | Main Window Structure

主窗口 HTML 分为三栏布局：

```
┌──────────────────────────────────────────────────────────┐
│  Left Sidebar (.sidebar)                                  │
│  ┌───────────────────────────────────────────────────┐   │
│  │  #agentList — Agent/Group 列表                    │   │
│  │  Tab: Topics / Settings                           │   │
│  │  #tabContentTopics — 话题列表                     │   │
│  │  #tabContentSettings — Agent 设置表单             │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  Center Area (.chat-area)                                │
│  ┌───────────────────────────────────────────────────┐   │
│  │  #chatHeader — 当前 Agent/Group 名称              │   │
│  │  #chatMessages — 消息列表                        │   │
│  │  #attachmentPreviewArea — 附件预览                │   │
│  │  #inputArea — 输入框 + 发送按钮                   │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  Right Sidebar (#notificationsSidebar)                   │
│  ┌───────────────────────────────────────────────────┐   │
│  │  VCP Log 连接状态                                 │   │
│  │  #notificationsList — 通知/日志列表               │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## 4. 安全模型 | Security Model

| 机制 Mechanism | 说明 Description |
|---------------|-----------------|
| `contextIsolation: true` | Renderer 无法访问 Node.js API，只能通过 contextBridge |
| `nodeIntegration: false` | 渲染进程完全沙箱化 |
| 白名单 IPC 频道 Whitelisted IPC channels | preload.js 中硬编码允许的频道列表 |
| `webSecurity: true` (默认) | 同源策略默认启用 |
| `protocol.registerFileProtocol` | 自定义协议 `vcpchat-file://` 用于安全访问本地文件 |

---

*→ 下一章: [03_modules_core.md](./03_modules_core.md)*  
*← 上一章: [01_overview.md](./01_overview.md)*
