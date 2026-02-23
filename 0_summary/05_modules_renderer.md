# 05 — 渲染子模块 | Renderer Sub-modules (`modules/renderer/`)

## 概述 | Overview

`modules/renderer/` 目录包含 `messageRenderer.js` 拆分出来的子模块，均以 **ES Module** 格式编写（`export`/`import`），仅在渲染进程中运行。

The `modules/renderer/` directory contains sub-modules split from `messageRenderer.js`, all written as **ES Modules** (`export`/`import`), running only in the renderer process.

---

## 1. `colorUtils.js` — 颜色工具 | Color Utilities

**职责 / Responsibilities:**

- 从 Agent 头像图片提取主色调（用于气泡渐变背景）
- 缓存颜色提取结果（`avatarColorCache` Map）

**关键 API:**

```javascript
export const avatarColorCache = new Map()

export async function getDominantAvatarColor(imageUrl)
// 返回 { r, g, b, hex } 或 null
// 通过 Canvas 像素分析提取主色
```

**使用场景 / Usage:**

每条 AI 消息气泡的背景渐变色从头像主色动态生成，让每个 Agent 拥有独特的视觉标识。

---

## 2. `enhancedColorUtils.js` — 增强颜色工具 | Enhanced Color Utilities

**职责 / Responsibilities:**

在 `colorUtils.js` 基础上提供更丰富的颜色操作：

- 颜色混合（blending）
- 对比度调整（contrast adjustment）
- 亮度/暗度计算（lightness/darkness）
- 颜色转换（RGB ↔ HSL ↔ HEX）

---

## 3. `animation.js` — 动画处理 | Animation Handling

**职责 / Responsibilities:**

处理消息气泡中的动画内容（通常是 AI 使用 VCP 工具生成的特效）：

- 解析并执行 Anime.js 动画描述符
- 处理 CSS 动画注入
- 管理 Three.js 3D 场景（如骰子动画、粒子效果）

**关键 API:**

```javascript
export function processAnimationsInContent(container, content)
// 在 container 中查找并激活动画代码块

export function cleanupAnimationsInContent(container)
// 清理动画资源（Three.js renderer 等）
```

---

## 4. `domBuilder.js` — DOM 构建器 | DOM Builder

**职责 / Responsibilities:**

创建标准化的消息骨架 DOM 结构，供 `messageRenderer.js` 填充内容。

**关键 API:**

```javascript
export function createMessageSkeleton(options)
// options: { role, agentName, avatarUrl, timestamp, msgId, ... }
// 返回: HTMLElement (完整的气泡骨架，含头像、名称、内容区域)
```

**气泡结构 / Bubble Structure:**

```html
<div class="message [user|assistant|tool-result]" data-msg-id="...">
  <div class="message-avatar">
    <img src="avatarUrl" />
  </div>
  <div class="message-content-wrapper">
    <div class="message-header">
      <span class="message-sender-name">AgentName</span>
      <span class="message-timestamp">HH:mm:ss</span>
    </div>
    <div class="message-bubble">
      <!-- 消息内容 / Message content -->
    </div>
    <div class="message-actions">
      <!-- 操作按钮（复制、重新生成、TTS 等）/ Action buttons -->
    </div>
  </div>
</div>
```

---

## 5. `contentProcessor.js` — 内容处理器 | Content Processor

**职责 / Responsibilities:**

这是消息渲染的核心处理链，将原始 AI 文本转换为富媒体 HTML：

1. **工具调用块处理** — 匹配 `TOOL_REGEX`，渲染为可展开的工具调用卡片
2. **日记块处理** — 匹配 `NOTE_REGEX`，渲染为日记卡片
3. **工具结果块处理** — 匹配 `TOOL_RESULT_REGEX`，渲染为结果折叠区
4. **按钮块处理** — 匹配 `BUTTON_CLICK_REGEX`，渲染为可点击按钮
5. **Canvas 占位符处理** — 匹配 `{{VCPChatCanvas}}`，渲染为打开 Canvas 的按钮
6. **思考链处理** — 匹配 `<think>` 和 VCP 元思考链标记，渲染为折叠的思考区域
7. **Markdown 渲染** — marked.js + DOMPurify 处理剩余内容
8. **数学公式渲染** — KaTeX 处理 `$...$` 和 `$$...$$`
9. **代码高亮** — highlight.js 处理代码块
10. **Mermaid 图表** — 检测并渲染 mermaid 代码块

**关键 API:**

```javascript
export async function processMessageContent(rawText, options)
// options: { isStreaming, agentConfig, msgId, ... }
// 返回: HTMLElement (已渲染的内容区域)

export function processToolCallBlock(content)
// 将工具调用标记转为 HTML 卡片

export function processMarkdown(text)
// Markdown → HTML (marked + DOMPurify + highlight)
```

---

## 6. `streamManager.js` — 流式管理器 | Stream Manager

**职责 / Responsibilities:**

管理流式响应（Streaming）的 UI 状态：

- 维护正在流式传输的消息 ID 集合
- 实时更新流式消息气泡（使用 `morphdom` 最小化 DOM 变更，减少重绘）
- 流结束后触发完整渲染（启用动画、数学公式等）
- 管理"正在输入"指示器（typing indicator）

**关键 API:**

```javascript
export function startStreaming(msgId, container)
// 标记消息开始流式传输

export function updateStreamContent(msgId, chunk)
// 追加/更新流式内容块

export function endStreaming(msgId)
// 完成流式传输，触发最终渲染

export function isStreaming(msgId)
// 检查消息是否仍在流式传输
```

**性能优化 / Performance Optimization:**

使用 `morphdom` 库进行 DOM diffing，流式更新时只修改变化的 DOM 节点，避免整体重渲染带来的闪烁和滚动跳跃。

---

## 7. `imageHandler.js` — 图片处理器 | Image Handler

**职责 / Responsibilities:**

- 处理消息中的图片（Base64、URL、本地路径）
- 懒加载（Intersection Observer）
- 图片点击放大（触发 image-viewer 窗口）
- 图片加载失败占位
- 处理从 VCP 后端返回的 Base64 图片数据
- 处理 `vcpchat-file://` 协议 URL

**关键 API:**

```javascript
export function initializeImageHandler(options)
// 注入 electronAPI 等依赖

export function setContentAndProcessImages(container, htmlContent)
// 设置 innerHTML 并处理其中的所有图片
```

---

## 8. `emoticonUrlFixer.js` — 表情包 URL 修复器 | Emoticon URL Fixer

**职责 / Responsibilities:**

修复从 VCP 后端返回的表情包 URL，将相对路径或特殊协议转换为可在 Electron 中正常加载的本地 URL。

**关键 API:**

```javascript
export function fixEmoticonUrls(container)
// 遍历 container 中的所有 img 标签，修复表情包 URL
```

---

## 9. `messageContextMenu.js` — 消息右键菜单 | Message Context Menu

**职责 / Responsibilities:**

处理消息气泡的右键菜单（上下文菜单）：

- 复制消息文本/Markdown/HTML
- 重新生成消息
- 删除消息
- 编辑消息
- 导出消息图片（html2canvas）
- 分享消息到 Forum
- 中止当前回复

**关键 API:**

```javascript
export function initializeContextMenu(container, options)
export function showContextMenu(event, msgId, msgData)
export function hideContextMenu()
```

---

## 10. `middleClickHandler.js` — 中键点击处理器 | Middle-Click Handler

**职责 / Responsibilities:**

处理消息气泡的中键点击行为（可在全局设置中配置中键动作）：

- 快速复制消息
- 快速重新生成
- 快速删除
- 自定义动作（通过全局设置 `middleClickQuickAction` 配置）

**关键 API:**

```javascript
export function initializeMiddleClickHandler(container, globalSettings)
export function handleMiddleClick(event, msgId)
```

---

## 11. `visibilityOptimizer.js` — 可见性优化器 | Visibility Optimizer

**职责 / Responsibilities:**

使用 **Intersection Observer API** 实现消息列表的懒渲染：

- 仅渲染当前视口及附近的消息气泡
- 离开视口的消息气泡替换为占位符（保留高度）
- 进入视口时恢复完整渲染

**目的 / Purpose:**

聊天历史可能包含数百至数千条消息。懒渲染确保即使在历史很长的情况下，UI 仍保持流畅。

**关键 API:**

```javascript
export function initialize(container)
// 初始化 IntersectionObserver

export function observeMessage(element)
// 开始观察一个消息元素

export function unobserveMessage(element)
// 停止观察（消息被删除时调用）

export function refreshAll()
// 强制重新评估所有可见性
```

---

## 模块依赖关系 | Module Dependency Map

```
messageRenderer.js
  │
  ├── imports colorUtils.js          (头像颜色提取)
  ├── imports enhancedColorUtils.js  (颜色计算)
  ├── imports animation.js           (动画处理)
  ├── imports domBuilder.js          (DOM 骨架构建)
  ├── imports contentProcessor.js    (内容渲染处理链)
  ├── imports streamManager.js       (流式状态管理)
  ├── imports imageHandler.js        (图片处理)
  ├── imports emoticonUrlFixer.js    (表情包 URL)
  ├── imports messageContextMenu.js  (右键菜单)
  ├── imports middleClickHandler.js  (中键动作)
  └── imports visibilityOptimizer.js (懒渲染优化)
```

---

*→ 下一章: [06_feature_modules.md](./06_feature_modules.md)*  
*← 上一章: [04_modules_ipc.md](./04_modules_ipc.md)*
