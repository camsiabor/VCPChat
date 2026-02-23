# 03 — 核心业务模块 | Core Business Modules (`modules/`)

## 概述 | Overview

`modules/` 目录包含渲染进程（及少量主进程共享）的核心业务逻辑。主要分为三类：

The `modules/` directory contains the core business logic for the renderer process (and a few shared with the main process). It falls into three categories:

1. **主渲染模块** — `chatManager.js`, `messageRenderer.js`, `vcpClient.js` 等
2. **IPC 处理层** — `modules/ipc/`（→ 详见 [04_modules_ipc.md](./04_modules_ipc.md)）
3. **渲染子模块** — `modules/renderer/`（→ 详见 [05_modules_renderer.md](./05_modules_renderer.md)）
4. **工具类** — `modules/utils/`

---

## 1. 聊天核心 | Chat Core

### `chatManager.js`

**进程**: 渲染进程 | **Pattern**: IIFE 模块，暴露为 `window.chatManager`

**核心职责 / Core Responsibilities:**

- 构建发送消息的上下文（附件、历史裁剪、系统提示注入、正则预处理）
- 接收流式响应并推入 `messageRenderer`
- 保存/加载聊天历史
- 管理消息的删除、重新生成
- 管理中止（interrupt）逻辑

**关键方法 / Key Methods:**

```javascript
chatManager.init(deps)                      // 注入依赖（electronAPI, uiHelper, 等）
chatManager.sendMessage(options)            // 发送消息到 VCP
chatManager.loadChatHistory(agentId, topicId) // 加载历史
chatManager.saveChatHistory()               // 保存历史
chatManager.deleteMessage(msgId)            // 删除消息
chatManager.regenerateMessage(msgId)        // 重新生成消息
chatManager.applyRegexRules(text, rules, scope, role, depth) // 正则处理
```

**正则处理流程 / Regex Processing Flow:**

```
用户输入文本
  │
  ▼
applyRegexRules(text, rules, 'frontend', 'user')
  │  过滤作用域为 frontend 的规则
  ▼
发送前上下文处理
  │
  ▼
applyRegexRules(text, rules, 'context', 'assistant', depth)
  │  过滤作用域为 context 的规则（历史压缩时）
  ▼
最终消息数组
```

---

### `vcpClient.js`

**进程**: 主进程（由 `modules/ipc/chatHandlers.js` 和 `modules/ipc/groupChatHandlers.js` 调用）

**核心职责 / Core Responsibilities:**

- 统一 VCP HTTP 请求（`POST /v1/chat/completions`）
- 处理流式响应（SSE / chunked transfer）
- 管理中止控制器（`AbortController` Map，按 `messageId` 索引）
- 将流式数据块通过 `webContents.send(streamChannel, chunk)` 推给渲染进程

**关键 API:**

```javascript
vcpClient.initialize(config)                // 注入 APP_DATA_ROOT, getMusicState
vcpClient.sendToVCP(params)                 // 主要请求函数
vcpClient.abortRequest(messageId)           // 中止指定请求
```

**`sendToVCP` 参数结构:**

```javascript
{
    vcpUrl,         // VCP 服务器 URL
    vcpApiKey,      // API 密钥
    messages,       // 消息数组 [{role, content}]
    modelConfig,    // { model, temperature, max_tokens, ... }
    messageId,      // 唯一消息 ID（用于中止）
    context,        // { agentId, topicId }
    webContents,    // 主窗口 webContents（用于推送）
    streamChannel,  // IPC 频道名称（默认 'vcp-stream-event'）
    onStreamEnd     // 回调函数
}
```

---

### `contextSanitizer.js`

**进程**: 主进程（被 chatHandlers, groupChatHandlers 调用）

**核心职责**: 清理发送给 VCP 的消息上下文，移除无效内容、处理多模态内容数组、防止发送超大消息。

---

## 2. 消息渲染 | Message Rendering

### `messageRenderer.js`

**进程**: 渲染进程 | **Pattern**: ES Module（使用 `import`）

**核心职责 / Core Responsibilities:**

- 将聊天历史和流式响应渲染为 DOM 气泡
- 解析并渲染各种特殊内容块：工具调用、日记、工具结果、按钮、Canvas 占位符
- Markdown 渲染（marked.js + DOMPurify + highlight.js）
- 数学公式渲染（KaTeX）
- Mermaid 图表渲染
- 3D 骰子动画
- 图片/视频/音频内嵌渲染
- 流式打字动画
- 可见性优化（懒渲染）

**关键正则常量 / Key Regex Constants:**

```javascript
TOOL_REGEX           // <<<[TOOL_REQUEST]>>>..<<<[END_TOOL_REQUEST]>>>
NOTE_REGEX           // <<<DailyNoteStart>>>...<<<DailyNoteEnd>>>
TOOL_RESULT_REGEX    // [[VCP调用结果信息汇总:...VCP调用结果结束]]
BUTTON_CLICK_REGEX   // [[点击按钮:...]]
CANVAS_PLACEHOLDER_REGEX  // {{VCPChatCanvas}}
THOUGHT_CHAIN_REGEX  // [--- VCP元思考链...][--- 元思考链结束 ---]
CONVENTIONAL_THOUGHT_REGEX  // <think>...</think>
```

**依赖的子模块 / Depends on sub-modules:**

```javascript
import { getDominantAvatarColor } from './renderer/colorUtils.js'
import { initializeImageHandler, setContentAndProcessImages } from './renderer/imageHandler.js'
import { processAnimationsInContent } from './renderer/animation.js'
import * as visibilityOptimizer from './renderer/visibilityOptimizer.js'
import { createMessageSkeleton } from './renderer/domBuilder.js'
import * as streamManager from './renderer/streamManager.js'
import * as emoticonUrlFixer from './renderer/emoticonUrlFixer.js'
import * as contentProcessor from './renderer/contentProcessor.js'
import * as contextMenu from './renderer/messageContextMenu.js'
import * as middleClickHandler from './renderer/middleClickHandler.js'
```

---

## 3. UI 管理层 | UI Management Layer

### `uiManager.js`

**进程**: 渲染进程

负责高层 UI 状态协调：显示/隐藏面板、切换视图、管理加载状态。

### `ui-helpers.js`

**进程**: 渲染进程 | **暴露为**: `window.uiHelperFunctions`

提供工具函数：

```javascript
uiHelperFunctions.regexFromString(pattern)       // 将字符串解析为 RegExp 对象
uiHelperFunctions.escapeHtml(text)               // HTML 转义
uiHelperFunctions.formatTimestamp(ts)            // 时间戳格式化
uiHelperFunctions.debounce(fn, delay)            // 防抖
uiHelperFunctions.throttle(fn, interval)         // 节流
```

### `event-listeners.js`

集中注册 DOM 事件监听器（键盘快捷键、窗口大小变化、拖拽等），避免在 `renderer.js` 中堆积大量事件绑定。

### `inputEnhancer.js`

增强输入框功能：自动高度调整、`@` 提及补全、快捷键（Shift+Enter 换行，Enter 发送）、粘贴图片。

---

## 4. Agent/Group 管理 | Agent & Group Management

### `itemListManager.js`

**进程**: 渲染进程

管理左侧边栏的 Agent/Group 列表 (`#agentList`)：

- 加载并渲染列表项（头像、名称、未读计数）
- 处理列表项点击（切换当前选中）
- 支持拖拽排序（Sortable.js）
- 管理未读消息计数角标

### `settingsManager.js`

**进程**: 渲染进程

管理右侧设置面板（选中 Agent 或 Group 时显示）：

- 填充并保存 Agent 设置表单（名称、提示词、模型参数、TTS 配置、正则规则）
- 管理 Group 设置表单（群成员、发言模式、邀请提示词）
- URL 补全（将裸 URL 补全为 `/v1/chat/completions`）
- 集成 `promptManager`（系统提示词编辑器）

### `topicListManager.js`

**进程**: 渲染进程

管理话题标签页 (`#tabContentTopics`)：

- 渲染话题列表（标题、日期、锁定状态）
- 话题的创建、重命名、删除、导出
- 拖拽排序
- 话题搜索过滤

---

## 5. 文件与媒体 | File & Media

### `fileManager.js`

**进程**: 主进程（被 IPC 处理器调用）

核心职责：基于内容寻址（SHA-256 哈希）的**中心化附件存储**。

```javascript
initializeFileManager(userDataPath, agentDataPath)
storeFile(sourcePathOrBuffer, originalName, agentId, topicId, fileTypeHint)
// 返回: { internalPath, hash, fileName, mimeType }

getAttachmentServeUrl(internalPath)   // 生成 vcpchat-file:// 协议 URL
extractTextFromFile(filePath, mime)   // 提取文档文本（PDF/DOCX/图片 OCR）
```

**内容寻址存储 / Content-addressed storage:**

```
AppData/UserData/attachments/{sha256hash}{ext}
```

相同内容只存一份，避免重复。

### `lyricFetcher.js`

**进程**: 渲染进程

从网络获取歌词（网易云音乐 API 等），供音乐播放器显示。

### `musicScannerWorker.js`

**进程**: Worker Thread（主进程启动）

扫描本地音乐文件夹，提取元数据（music-metadata），运行在独立线程避免阻塞主进程。

---

## 6. 搜索与过滤 | Search & Filter

### `searchManager.js`

**进程**: 渲染进程

基于 FlexSearch 的全文搜索：

- 建立消息内容索引
- 跨话题搜索消息
- 高亮搜索结果

### `filterManager.js`

**进程**: 渲染进程

内容过滤器：根据用户配置的规则（正则/关键词）过滤/隐藏/替换消息内容。

---

## 7. 通知与状态 | Notifications & Status

### `notificationRenderer.js`

**进程**: 渲染进程

管理右侧通知面板：

- 接收来自主进程的 VCP 日志推送
- 渲染通知列表（工具调用结果、系统事件、异步任务完成）
- 系统 OS 通知（`Notification` API）

### `modelUsageTracker.js`

**进程**: 渲染进程

追踪各模型的使用频次，用于"热门模型"排序功能。

---

## 8. 其他工具模块 | Other Utility Modules

### `interruptHandler.js`

管理用户中止 AI 回复的逻辑：调用 `vcpClient.abortRequest(messageId)` 并清理相关 UI 状态。

### `emoticonManager.js`

管理表情包：从本地或 VCP 后端加载表情包列表，支持搜索和插入。

### `topicSummarizer.js`

使用 AI 自动生成话题标题：

```javascript
summarizeTopicFromMessages(messages, agentName)
// 当对话达到 4+ 条消息时，调用 VCP API 生成 10 字以内的标题
```

默认模型：`gemini-2.5-flash`（可在设置中覆盖）。

### `speechRecognizer.js`

**懒加载**（按需 require）。封装 Web Speech API 或系统语音识别能力，用于语音输入。

### `SovitsTTS.js`

封装 SoVITS TTS API 调用，用于将 AI 回复文本转为语音播放。

### `global-settings-manager.js`

管理"全局设置"弹窗（服务器 URL、API Key、用户名、主题、TTS 全局配置等）。

---

## 9. 工具类 | Utilities (`modules/utils/`)

### `agentConfigManager.js`

**进程**: 主进程

提供线程安全的 Agent 配置读写操作（基于文件锁或队列，防止并发写入损坏 JSON）：

```javascript
agentConfigManager.getAgentConfig(agentId)
agentConfigManager.updateAgentConfig(agentId, updaterFn)  // 原子更新
agentConfigManager.saveAgentConfig(agentId, config)
```

### `appSettingsManager.js`

**进程**: 主进程

管理全局 `settings.json` 的读写，提供类似 `agentConfigManager` 的安全访问层。

---

## 10. 查看器窗口 | Viewer Windows

### `modules/image-viewer.html` + `modules/image-viewer.js`

独立的图片查看器子窗口。支持：缩放、旋转、全屏、下载。

### `modules/text-viewer.html` + `modules/text-viewer.js`

独立的文本/代码查看器子窗口。支持：语法高亮、行号、复制。

---

*→ 下一章: [04_modules_ipc.md](./04_modules_ipc.md)*  
*← 上一章: [02_electron_core.md](./02_electron_core.md)*
