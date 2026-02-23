# 10 — UI 样式体系与主题系统 | UI Styling System & Theming

## 概述 | Overview

VCPChat 的样式体系经历了从单文件 (`style.css`, `themes.css`) 到模块化 CSS 的演进，当前以 `styles/` 目录为主要样式源。

VCPChat's styling has evolved from single files (`style.css`, `themes.css`) to modular CSS; the `styles/` directory is the current primary style source.

---

## 1. CSS 文件结构 | CSS File Structure

### 1.1 `styles/` — 模块化样式（当前主力）| Modular Styles (Current Primary)

```
styles/
├── base.css              # 基础重置、CSS 变量定义、字体 / Base reset, CSS vars, fonts
├── layout.css            # 整体布局（三栏结构、调整大小器）/ Overall layout (3-column)
├── chat.css              # 聊天区域（消息列表、输入框）/ Chat area
├── components.css        # 通用组件（按钮、模态框、下拉）/ Common components
├── settings.css          # 设置面板样式 / Settings panel styles
├── search.css            # 搜索界面 / Search UI
├── notifications.css     # 通知侧边栏 / Notifications sidebar
├── animations.css        # 过渡动画和关键帧 / Transitions and keyframes
├── messageRenderer.css   # 消息气泡专属（工具调用卡片、日记、代码块等）
└── themes.css            # 主题变量（CSS 自定义属性）/ Theme variables (CSS custom props)
```

### 1.2 遗留文件 | Legacy Files

```
style.css          # 遗留全局样式（仍被 main.html 引用，与 styles/ 并存）
themes.css         # 遗留主题文件（被 styles/themes.css 取代）
```

---

## 2. CSS 变量系统 | CSS Variable System

所有颜色、间距、圆角等均通过 CSS 自定义属性（变量）定义，集中在 `styles/themes.css` 中：

```css
/* 深色主题默认值 / Dark theme defaults */
:root {
    /* 主色调 / Primary colors */
    --primary-bg: #1a1a2e;
    --secondary-bg: #16213e;
    --panel-bg: rgba(20, 20, 30, 0.85);
    --primary-text: #e0e0e0;
    --secondary-text: #a0a0b0;
    --accent-color: #7b68ee;
    --accent-hover: #9b88ff;

    /* 气泡颜色 / Bubble colors */
    --user-bubble-bg: rgba(123, 104, 238, 0.15);
    --assistant-bubble-bg: rgba(30, 30, 50, 0.9);

    /* 侧边栏 / Sidebar */
    --sidebar-bg: rgba(15, 15, 25, 0.95);
    --sidebar-item-hover: rgba(123, 104, 238, 0.1);
    --sidebar-item-selected: rgba(123, 104, 238, 0.25);

    /* 边框 / Borders */
    --border-color: rgba(255, 255, 255, 0.08);
    --divider-color: rgba(255, 255, 255, 0.05);

    /* 毛玻璃效果 / Frosted glass */
    --glass-bg: rgba(20, 20, 35, 0.7);
    --glass-blur: blur(20px);
    --glass-border: 1px solid rgba(255, 255, 255, 0.1);

    /* 代码块 / Code blocks */
    --code-bg: #0d1117;
    --code-border: rgba(255, 255, 255, 0.1);
}

/* 亮色主题覆盖 / Light theme override */
body.light-theme {
    --primary-bg: #f0f2f5;
    --secondary-bg: #ffffff;
    --panel-bg: rgba(255, 255, 255, 0.9);
    --primary-text: #1a1a2e;
    /* ... */
}
```

---

## 3. 主题系统 | Theming System

### 3.1 内置主题 | Built-in Themes

| 主题 Theme | body class | 描述 |
|-----------|-----------|------|
| Dark（默认）| (无/no class) | 深蓝紫色深色主题 |
| Light | `light-theme` | 明亮白色主题 |
| Sakura Night | `sakura-theme` | 樱花夜间主题 |

### 3.2 主题切换机制 | Theme Switch Mechanism

```javascript
// 渲染进程触发
window.electronAPI.setTheme('dark' | 'light' | 'sakura')

// 主进程 themeHandlers.js 处理
// 通过 webContents.send 通知渲染进程
// 渲染进程修改 document.body.className
```

### 3.3 壁纸系统 | Wallpaper System

```javascript
// 壁纸存储在用户自定义目录中
// 主进程扫描并生成缩略图（sharp 库）
// 保存于 AppData/WallpaperThumbnailCache/

// 设置壁纸
document.body.style.backgroundImage = `url("vcpchat-file://${wallpaperPath}")`
```

### 3.4 自定义 CSS | Custom CSS

每个 Agent 可设置独立的 `customCss` 字段，在该 Agent 的聊天界面中注入自定义样式：

```javascript
// settings.json (agent config)
{
    "customCss": ".message-bubble { border-radius: 20px; }"
}

// 切换到 Agent 时，渲染进程注入
const styleEl = document.createElement('style')
styleEl.textContent = agentConfig.customCss
document.head.appendChild(styleEl)
```

---

## 4. 消息气泡样式 | Message Bubble Styles

消息气泡由 `messageRenderer.css` 控制，核心样式：

```
┌─────────────────────────────────────────────┐
│ .message                                     │
│  ├── .message-avatar (圆形头像)              │
│  └── .message-content-wrapper               │
│       ├── .message-header                   │
│       │    ├── .message-sender-name         │
│       │    └── .message-timestamp           │
│       ├── .message-bubble                   │
│       │    ├── 普通文本 / Markdown           │
│       │    ├── .tool-call-card (工具调用)    │
│       │    ├── .daily-note-card (日记)       │
│       │    ├── .thought-chain (思考链)       │
│       │    ├── .code-block (代码块)          │
│       │    └── .mermaid-diagram (图表)       │
│       └── .message-actions                  │
│            ├── 复制按钮 / Copy button        │
│            ├── TTS 按钮 / TTS button         │
│            └── 重新生成 / Regenerate         │
└─────────────────────────────────────────────┘
```

### 动态头像色 | Dynamic Avatar Color

每个 Agent 头像主色会被提取，用于气泡背景渐变：

```css
.message-bubble {
    background: linear-gradient(
        135deg,
        rgba(var(--avatar-color-r), var(--avatar-color-g), var(--avatar-color-b), 0.15),
        var(--assistant-bubble-bg)
    );
}
```

---

## 5. 动画系统 | Animation System

`styles/animations.css` 定义核心关键帧动画：

```css
/* 消息进入动画 / Message entry animation */
@keyframes messageSlideIn {
    from { opacity: 0; transform: translateY(10px); }
    to   { opacity: 1; transform: translateY(0); }
}

/* 流式光标 / Streaming cursor */
@keyframes typingCursor {
    0%, 100% { opacity: 1; }
    50% { opacity: 0; }
}

/* 心流锁发光效果 / Flow lock glow effect (in flowlock.css) */
@keyframes flowlockPulse {
    0%, 100% { text-shadow: 0 0 5px var(--accent-color); }
    50% { text-shadow: 0 0 20px var(--accent-color), 0 0 40px var(--accent-hover); }
}
```

---

## 6. 子窗口样式 | Child Window Styles

各功能子窗口有独立的 CSS 文件，同时引用 `styles/themes.css` 获取主题变量：

```
Musicmodules/music.css
Canvasmodules/canvas.css
Forummodules/forum.css
Flowlockmodules/flowlock.css
Memomodules/memo.css
Dicemodules/dice.css
Voicechatmodules/voicechat.css
Translatormodules/translator.css
Themesmodules/themes-module.css
Promptmodules/prompt-modules.css
Assistantmodules/assistant.css
Notemodules/notes.css
```

---

## 7. Vendor 样式库 | Vendor Style Libraries

```
vendor/
├── atom-one-dark.min.css    # highlight.js 深色代码主题
├── atom-one-light.min.css   # highlight.js 亮色代码主题
└── katex.min.css            # KaTeX 数学公式样式
```

---

*→ 下一章: [11_companion_tools.md](./11_companion_tools.md)*  
*← 上一章: [09_data_storage.md](./09_data_storage.md)*
