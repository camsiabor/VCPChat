# 08 — VCP 协议与后端通信 | VCP Protocol & Backend Communication

## 1. VCP 协议简介 | VCP Protocol Introduction

**VCP（Variable & Command Protocol，变量与命令协议）** 是 VCPChat 配套后端 VCPToolBox 定义的 AI 工具调用协议。其核心设计哲学是：**将 AI 视为创造者伙伴，使用文本标记而非严格 JSON 进行工具调用，兼容所有 LLM**。

**VCP (Variable & Command Protocol)** is the AI tool-call protocol defined by VCPToolBox (VCPChat's companion backend). Its core philosophy: **treat AI as a creative partner, use text markers instead of strict JSON for tool calls, compatible with all LLMs**.

---

## 2. 工具调用语法 | Tool Call Syntax

### 2.1 工具请求标记 | Tool Request Marker

```
<<<[TOOL_REQUEST]>>>
tool_name:「始」ToolName「末」,
param1:「始」value1「末」,
param2:「始」value2「末」
<<<[END_TOOL_REQUEST]>>>
```

- 开始标记：`<<<[TOOL_REQUEST]>>>`
- 结束标记：`<<<[END_TOOL_REQUEST]>>>`
- 参数格式：`key:「始」value「末」`（使用日文引号作为值界定符，避免与普通文本冲突）
- AI 可在一次回复中包含**多个**工具调用块（并行执行）

### 2.2 工具结果标记 | Tool Result Marker

```
[[VCP调用结果信息汇总:
工具: ToolName
结果: ...
VCP调用结果结束]]
```

### 2.3 日记标记 | Diary Marker

```
<<<DailyNoteStart>>>
{日记内容}
<<<DailyNoteEnd>>>
```

### 2.4 按钮标记 | Button Marker

```
[[点击按钮:ButtonLabel]]
```

渲染为可点击按钮，点击后将按钮文本作为用户消息发送。

### 2.5 Canvas 占位符 | Canvas Placeholder

```
{{VCPChatCanvas}}
```

渲染为"打开 Canvas"按钮。

### 2.6 思考链标记 | Thought Chain Markers

```
[--- VCP元思考链: "思考主题" ---]
{思考内容}
[--- 元思考链结束 ---]
```

或标准格式：

```
<think>
{思考内容}
</think>
```

---

## 3. VCP 服务器通信 | VCP Server Communication

### 3.1 Chat Completions API

VCPChat 通过标准 **OpenAI 兼容** 的 Chat Completions 端点与 VCP 服务器通信：

```
POST {vcpServerUrl}/v1/chat/completions
Authorization: Bearer {vcpApiKey}
Content-Type: application/json

{
    "model": "model-name",
    "messages": [...],
    "temperature": 0.7,
    "max_tokens": 32000,
    "stream": true
}
```

- URL 由用户在全局设置中配置，`settingsManager.js` 的 `completeVcpUrl()` 函数自动补全为 `/v1/chat/completions`。
- 支持流式响应（SSE/chunked transfer）。
- `vcpClient.js` 中的 `AbortController` Map 管理中止。

### 3.2 模型列表 API

```
GET {vcpServerUrl}/v1/models
Authorization: Bearer {vcpApiKey}
```

用于刷新可用模型列表（主进程缓存在 `cachedModels` 数组中）。

### 3.3 VCP Log WebSocket

```
WebSocket: {vcpLogUrl}/vcpinfo/VCP_Key={vcpApiKey}
```

- 接收 VCP 后端的实时日志推送
- 接收异步工具调用结果
- 主进程维持连接，自动断线重连（最长退避 60 秒）
- 收到消息后通过 `webContents.send('vcp-notification', data)` 推给渲染进程

---

## 4. 分布式服务器通信 | Distributed Server Communication

`VCPDistributedServer` 作为 VCPChat 内嵌的分布式节点，与主 VCP 服务器建立 WebSocket 连接：

```
WebSocket: {mainServerUrl}/?VCP_Key={vcpKey}&serverName={name}
```

该节点可：
- 接收来自 VCP 主服务器的指令（控制骰子、Canvas、Flowlock、音乐等）
- 提供本地 HTTP 静态文件服务
- 管理本地插件（`Plugin.js`）

---

## 5. 上下文构建流程 | Context Building Flow

发送消息前，`chatHandlers.js` 会构建完整的消息上下文数组：

```
1. 系统提示词注入 System prompt injection
   └── 根据 Agent 的提示词模式（原始/模块化/预制）获取系统提示词
   └── 插入到消息数组开头（role: 'system'）

2. 历史消息加载 History loading
   └── 从 history.json 读取
   └── 按 contextTokenLimit 裁剪（从最新消息向前保留）

3. 世界书注入 World book injection
   └── 根据关键词匹配注入相关条目

4. 正则预处理 Regex pre-processing
   └── 对历史消息应用 'context' 作用域的正则规则

5. 多模态内容处理 Multimodal content processing
   └── 图片附件转为 Base64 或 URL（content 数组格式）
   └── 文档附件提取文本

6. 音乐状态注入 Music state injection（可选）
   └── 当前播放音乐信息注入到系统提示词

7. 发送 Send
   └── vcpClient.sendToVCP(...)
```

---

## 6. 流式响应处理 | Streaming Response Handling

```
HTTP 流式响应 (SSE chunks)
    │
    ▼
vcpClient.js: 逐块读取 response.body
    │
    ▼
解析 SSE 格式:
  "data: {json}\n\n"
  → JSON.parse → delta.content
    │
    ▼
webContents.send('vcp-stream-event', {
    type: 'chunk',
    content: deltaText,
    messageId: '...'
})
    │
    ▼
renderer.js 监听 'vcp-stream-event'
    │
    ▼
chatManager → messageRenderer.updateStreamContent(msgId, chunk)
    │
    ▼
streamManager.js → morphdom DOM 更新
    │
    ▼
流结束 (finish_reason: 'stop')
    │
    ▼
webContents.send('vcp-stream-event', { type: 'end', messageId })
    │
    ▼
messageRenderer 完整渲染（解析工具调用、Markdown、数学公式等）
```

---

## 7. 中止机制 | Abort Mechanism

```javascript
// 渲染进程触发中止
window.electronAPI.abortChatMessage(messageId)

// 主进程处理
ipcMain.handle('abort-chat-message', (event, messageId) => {
    vcpClient.abortRequest(messageId)  // 调用 AbortController.abort()
    // 同时终止相关 VCP 工具调用进程（通知 VCP 服务器）
})
```

---

## 8. 多模态支持 | Multimodal Support

### 图片传输 / Image Transmission

```javascript
// 发送图片时，content 格式为数组：
messages = [{
    role: 'user',
    content: [
        { type: 'text', text: '请描述这张图片' },
        { type: 'image_url', image_url: { url: 'data:image/png;base64,...' } }
    ]
}]
```

### Base64 直通车 / Base64 Direct Passthrough

VCP 后端支持在 `tool` 字段中直接携带 Base64 数据，前端通过 `imageHandler.js` 直接渲染。

### VCPFileAPI

后端维护一个全局文件 URL 系统，允许任意节点通过文件路径跨节点访问文件：

```
VCPFileAPI v4.0: 全 URL 超栈追踪
任意节点文件路径 → 主服务器智能解析来源 → 自动向源节点请求 Base64 数据
```

---

## 9. Agent 自主话题管理 | Agent Autonomous Topic Management

VCP 服务器暴露话题管理 API，允许 AI 自主操作聊天话题：

```
# AI 可通过工具调用执行：
- 读取话题列表
- 读取特定话题历史
- 创建新话题
- 重命名话题
- 切换到指定话题
```

这使得在后台运行的 Agent 能够主动发起新对话，无需用户手动干预。

---

## 10. 配置参数 | Configuration Parameters

全局设置 `settings.json` 中与 VCP 通信相关的字段：

```json
{
    "vcpServerUrl": "http://localhost:3000/v1/chat/completions",
    "vcpApiKey": "your-api-key",
    "vcpLogUrl": "ws://localhost:5890",
    "vcpLogKey": "your-log-key",
    "topicSummaryModel": "gemini-2.5-flash",
    "enableThoughtChainInjection": false,
    "flowlockContinueDelay": 5
}
```

每个 Agent 的 `config.json` 包含：

```json
{
    "vcpServerUrl": "",         // 可覆盖全局 URL
    "model": "gpt-4o",
    "temperature": 0.7,
    "contextTokenLimit": 128000,
    "maxOutputTokens": 32000,
    "systemPrompt": "...",
    "topics": [...]
}
```

---

*→ 下一章: [09_data_storage.md](./09_data_storage.md)*  
*← 上一章: [07_audio_engine.md](./07_audio_engine.md)*
