# 04 — IPC 处理层 | IPC Handler Layer (`modules/ipc/`)

## 概述 | Overview

`modules/ipc/` 目录包含**主进程端**的所有 IPC（Inter-Process Communication）处理器。每个文件对应一个功能域，导出一个 `initialize(mainWindow, context)` 函数，在 `main.js` 启动时统一注册。

The `modules/ipc/` directory contains all **main-process-side** IPC handlers. Each file corresponds to a functional domain and exports an `initialize(mainWindow, context)` function, registered together during `main.js` startup.

---

## IPC 调用模式 | IPC Call Patterns

Electron IPC 有三种调用模式，本项目均有使用：

```
1. ipcMain.handle(channel, handler)    ← ipcRenderer.invoke(channel, data)
   请求-响应模式（async/await）/ Request-response (async/await)

2. ipcMain.on(channel, handler)        ← ipcRenderer.send(channel, data)
   单向发送 / One-way send

3. webContents.send(channel, data)     → ipcRenderer.on(channel, callback)
   主进程主动推送 / Main process push
```

---

## 各处理器文件详解 | Handler Files in Detail

### `agentHandlers.js`

管理 Agent 配置的 CRUD 操作。

**注册的频道 / Registered channels:**

```
get-agents              → 获取所有 Agent 列表（含配置）
get-agent-config        → 获取单个 Agent 配置
save-agent-config       → 保存 Agent 配置
create-agent            → 创建新 Agent（生成 UUID，创建目录）
delete-agent            → 删除 Agent（删除目录和配置）
save-avatar             → 保存 Agent 头像（Base64 → PNG 文件）
select-avatar           → 打开文件选择对话框选择头像
update-agent-config     → 部分更新 Agent 配置（原子操作）
get-all-items           → 获取所有 Agent + Group 的摘要列表
get-global-warehouse    → 获取全局仓库数据
save-global-warehouse   → 保存全局仓库数据
get-hot-models          → 获取使用频率最高的模型
get-favorite-models     → 获取收藏的模型
toggle-favorite-model   → 切换模型收藏状态
get-cached-models       → 获取缓存的模型列表
import-regex-rules      → 从文件导入正则规则到 Agent
```

**数据存储路径 / Data storage path:**

```
AppData/Agents/{agentId}/
    config.json          ← Agent 配置
    avatar.png           ← Agent 头像（可选）
    topics/
        {topicId}/
            history.json ← 聊天历史
```

**`getAgentConfigById(agentId)`** 函数被导出为具名导出，供其他 IPC 处理器跨模块调用。

---

### `chatHandlers.js`

管理聊天历史和话题的读写操作。

**注册的频道 / Registered channels:**

```
get-chat-history          → 读取话题聊天历史 (history.json)
save-chat-history         → 保存话题聊天历史（含文件监听信号）
delete-message            → 从历史中删除指定消息
save-topic-order          → 保存 Agent 话题排序
save-group-topic-order    → 保存 Group 话题排序
get-unread-topic-counts   → 获取未读话题计数
update-topic-lock         → 更新话题锁定状态
export-topic-markdown     → 导出话题为 Markdown 文件
export-topic-html         → 导出话题为 HTML 文件
search-messages           → 全文搜索消息（跨话题）
get-agent-chat-stats      → 获取 Agent 聊天统计（消息数、话题数）
```

**依赖 / Dependencies:**

- `modules/contextSanitizer` — 消息内容清理
- `fileWatcher` — 文件修改监听
- `agentConfigManager` — 原子配置更新

---

### `groupChatHandlers.js`

管理群聊（Group Chat）的完整生命周期。

**注册的频道 / Registered channels:**

```
get-agent-groups          → 获取所有群组配置
create-new-group          → 创建群组
delete-group              → 删除群组
save-group-config         → 保存群组配置
get-group-chat-history    → 读取群聊历史
save-group-chat-history   → 保存群聊历史
get-group-topics          → 获取群组话题列表
create-group-topic        → 创建群组话题
save-group-topic-title    → 重命名群组话题
delete-group-topic        → 删除群组话题
send-group-message        → 群聊发言（触发多 Agent 轮流响应）
```

**数据存储路径 / Data storage path:**

```
AppData/AgentGroups/{groupId}/
    config.json            ← 群组配置（成员、发言模式等）
```

```
AppData/UserData/{groupId}/topics/{topicId}/
    history.json           ← 群聊历史
```

**委托给 / Delegates to:**

- `Groupmodules/groupchat.js` — 核心群聊逻辑

---

### `settingsHandlers.js`

管理全局应用设置。

**注册的频道 / Registered channels:**

```
load-settings            → 读取 settings.json
save-settings            → 写入 settings.json
save-user-avatar         → 保存用户头像
save-avatar-color        → 保存头像颜色（Agent 或用户）
get-agent-topics         → 获取 Agent 话题列表（创建/删除/重命名代理）
create-new-topic-for-agent → 为 Agent 创建新话题
save-agent-topic-title   → 重命名话题
delete-topic             → 删除话题
```

---

### `windowHandlers.js`

管理窗口操作（最小化、最大化、关闭、拖动等）。

**注册的频道 / Registered channels:**

```
minimize-window          → 最小化窗口
maximize-window          → 最大化/还原窗口
close-window             → 关闭窗口
set-window-position      → 设置窗口位置
get-window-state         → 获取窗口状态
open-external-url        → 打开外部链接（shell.openExternal）
```

---

### `fileDialogHandlers.js`

管理文件对话框和附件操作。

**注册的频道 / Registered channels:**

```
select-file              → 打开文件选择对话框
save-attachment          → 存储附件（委托给 fileManager）
get-attachment-url       → 获取附件 URL（vcpchat-file:// 协议）
open-file-in-viewer      → 在查看器窗口中打开文件
open-image-in-viewer     → 在图片查看器中打开图片
open-text-in-viewer      → 在文本查看器中打开文本
save-user-avatar         → 保存用户头像文件
select-directory         → 选择目录
```

---

### `musicHandlers.js`

管理音乐播放器相关操作（与 Rust 音频引擎通信）。

**注册的频道 / Registered channels:**

```
open-music-window        → 打开/聚焦音乐窗口
music-load               → 加载音乐文件到引擎
music-play               → 播放
music-pause              → 暂停
music-seek               → 跳转
music-get-state          → 获取播放状态
music-set-volume         → 设置音量
music-get-devices        → 获取音频输出设备列表
music-configure-output   → 配置输出设备
music-set-eq             → 设置均衡器参数
music-set-eq-type        → 设置 EQ 类型（FIR/IIR）
music-configure-optimizations → 配置音质优化
music-configure-upsampling    → 配置升频设置
music-get-lyrics         → 获取已存储歌词
music-fetch-lyrics       → 从网络获取歌词
get-music-playlist       → 获取播放列表
save-music-playlist      → 保存播放列表
get-custom-playlists     → 获取自定义播放列表
save-custom-playlists    → 保存自定义播放列表
share-file-to-main       → 从音乐窗口分享文件到主窗口
```

---

### `canvasHandlers.js`

管理 Canvas 协同编辑器。

**注册的频道 / Registered channels:**

```
get-canvas-files         → 获取 Canvas 文件列表
open-canvas-window       → 打开 Canvas 窗口
save-canvas-file         → 保存文件内容
delete-canvas-file       → 删除文件
rename-canvas-file       → 重命名文件
run-canvas-python        → 在沙箱中运行 Python 代码
get-canvas-file-history  → 获取文件变更历史
canvas-file-updated      → [推送] 通知文件被外部更新
```

**数据存储 / Data storage:**

```
AppData/canvas/{filename}           ← 文件内容
AppData/canvas/.history/{filename}  ← 版本历史
```

---

### `assistantHandlers.js`

管理迷你助手侧边栏（浮动在桌面上方的 AI 助手）。

**包含 / Includes:**

- selection-hook 的初始化（监听全局文本选中事件）
- 助手窗口的创建和定位
- 将选中文本发送给 AI 处理

---

### `promptHandlers.js`

管理系统提示词（Preset、角色卡、世界书）。

**注册的频道 / Registered channels:**

```
load-preset-prompts       → 从目录加载预设提示词列表
load-preset-content       → 加载指定预设文件内容
get-active-system-prompt  → 获取当前 Agent 生效的系统提示词
programmatic-set-prompt-mode → 程序化切换提示词模式
```

---

### `notesHandlers.js`

管理笔记功能（本地笔记和网络笔记）。

**注册的频道 / Registered channels:**

```
open-notes-window         → 打开笔记窗口
get-network-notes         → 获取网络笔记树（带缓存）
save-network-notes-cache  → 保存网络笔记缓存
```

---

### `regexHandlers.js`

管理 Agent 正则规则的 CRUD。

**注册的频道 / Registered channels:**

```
get-agent-regex           → 获取 Agent 的正则规则列表
save-agent-regex          → 保存正则规则列表
test-regex                → 测试正则表达式（输入+规则→输出预览）
import-regex-from-file    → 从 JSON 文件导入规则
export-regex-to-file      → 导出规则到 JSON 文件
```

---

### `sovitsHandlers.js`

管理 SoVITS TTS（语音合成）集成。

**注册的频道 / Registered channels:**

```
get-sovits-models         → 获取可用 TTS 模型列表
test-sovits-tts           → 测试 TTS（生成并播放）
generate-tts              → 生成 TTS 音频（返回音频数据）
get-tts-config            → 获取 TTS 全局配置
save-tts-config           → 保存 TTS 全局配置
```

---

### `themeHandlers.js`

管理应用主题。

**注册的频道 / Registered channels:**

```
get-themes                → 获取内置主题列表
set-theme                 → 切换主题（通知渲染进程）
get-custom-theme          → 获取自定义主题 CSS
set-custom-theme          → 保存自定义主题 CSS
get-wallpapers            → 获取壁纸列表（含缩略图生成）
set-wallpaper             → 设置壁纸
```

---

### `emoticonHandlers.js`

管理表情包存储和检索。

**注册的频道 / Registered channels:**

```
get-emoticons             → 获取表情包列表（本地 + 缓存）
save-emoticon             → 保存表情包
delete-emoticon           → 删除表情包
search-emoticons          → 搜索表情包
```

---

### `forumHandlers.js`

管理论坛配置和窗口。

**注册的频道 / Registered channels:**

```
open-forum-window         → 打开论坛子窗口
get-forum-config          → 读取论坛配置（用户名、密码等）
save-forum-config         → 保存论坛配置
```

---

### `memoHandlers.js`

管理备忘录（快速记录卡片）。

**注册的频道 / Registered channels:**

```
open-memo-window          → 打开备忘录窗口
get-memos                 → 获取所有备忘录
save-memo                 → 保存单条备忘录
delete-memo               → 删除备忘录
```

---

### `diceHandlers.js`

管理骰子子窗口。

**注册的频道 / Registered channels:**

```
open-dice-window          → 打开骰子窗口
dice-roll                 → 骰子投掷结果广播
```

---

## 通用 Context 对象 | Common Context Object

所有处理器的 `initialize(mainWindow, context)` 接收的 `context` 对象包含：

```javascript
{
    AGENT_DIR,                    // AppData/Agents
    USER_DATA_DIR,                // AppData/UserData
    APP_DATA_ROOT_IN_PROJECT,     // AppData/
    NOTES_AGENT_ID,               // 'notes_attachments_agent'
    SETTINGS_FILE,                // AppData/settings.json
    getMusicState,                // 获取音乐播放状态的函数
    fileWatcher,                  // 文件监听器对象
    agentConfigManager,           // Agent 配置管理器
    getSelectionListenerStatus,   // 获取 selection-hook 状态
    stopSelectionListener,        // 停止 selection-hook
    startSelectionListener,       // 启动 selection-hook
    distributedServer,            // 分布式服务器实例
    ...
}
```

---

*→ 下一章: [05_modules_renderer.md](./05_modules_renderer.md)*  
*← 上一章: [03_modules_core.md](./03_modules_core.md)*
