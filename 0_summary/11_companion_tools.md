# 11 — 伴生工具 | Companion Tools

本章介绍 VCPChat 主应用之外的三个独立工具，它们各自是完整的 Electron 应用或独立模块。

This chapter covers the three standalone tools outside the main VCPChat application, each being a complete Electron app or independent module.

---

## 1. `VCPDistributedServer/` — 分布式节点服务器 | Distributed Node Server

### 概述 | Overview

`VCPDistributedServer` 是**嵌入在 VCPChat 主应用内**的轻量级分布式服务器节点。它在主进程中作为独立的 Node.js 模块运行，连接到 VCPToolBox 主服务器，使 VCPChat 客户端成为 VCP 分布式网络的一个节点。

`VCPDistributedServer` is a lightweight distributed server node **embedded inside the VCPChat main app**. It runs as an independent Node.js module in the main process, connecting to the VCPToolBox main server to make VCPChat a node in the VCP distributed network.

### 文件 | Files

```
VCPDistributedServer/
├── VCPDistributedServer.js    # DistributedServer 类实现
├── Plugin.js                  # 本地插件管理器
└── config.env                 # 配置（端口等）
```

`config.env`:
```env
DIST_SERVER_PORT=5974    # 分布式服务器 HTTP 端口
```

### 架构 | Architecture

```
VCPChat Main Process
    │
    └── DistributedServer 实例
         │
         ├── Express HTTP Server (localhost:5974)
         │    └── 静态文件服务、本地 API
         │
         ├── WebSocket Client → VCPToolBox 主服务器
         │    URL: {mainServerUrl}/?VCP_Key={vcpKey}&serverName={name}
         │    • 断线自动重连（指数退避，最大 60s）
         │    • 接收来自主服务器的控制指令
         │
         └── Plugin.js → 本地插件执行
```

### 核心功能 | Core Features

1. **注册为分布式节点**: 向 VCPToolBox 主服务器注册，成为可被 AI 感知的节点
2. **本地插件执行**: 执行 AI 发起的本地插件（通过 `Plugin.js`）
3. **控制媒体**: 接收并执行 AI 的音乐控制、骰子、Canvas、Flowlock 指令
4. **静态文件服务**: 为需要访问本地文件的 AI 工具提供文件服务
5. **静态占位符更新**: 定期更新注入到上下文的动态占位符（如当前时间、音乐状态等）

### 注入的控制器 | Injected Controllers

通过构造函数注入，将媒体控制函数传递给分布式服务器：

```javascript
new DistributedServer({
    mainServerUrl: settings.vcpServerUrl,
    vcpKey: settings.vcpApiKey,
    serverName: 'VCPChat',
    port: 5974,
    handleMusicControl: (cmd) => { ... },    // 音乐控制
    handleDiceControl: (cmd) => { ... },     // 骰子控制
    handleCanvasControl: (cmd) => { ... },   // Canvas 控制
    handleFlowlockControl: (cmd) => { ... }, // 心流锁控制
    rendererProcess: mainWindow.webContents  // 渲染进程通信
})
```

### `Plugin.js` — 本地插件管理器 | Local Plugin Manager

管理 VCPChat 本地插件的注册和执行：

- 扫描插件目录
- 按需执行插件脚本
- 插件遵循 VCP 插件规范（`plugin-manifest.json` + 实现文件）

---

## 2. `VchatManager/` — VChat 管理工具 | VChat Manager Tool

### 概述 | Overview

VchatManager 是一个**独立的 Electron 应用**（有自己的 `package.json` 和 `main.js`），专门用于管理 VCPChat 的数据，特别是 Agent 和群组的配置数据一致性检查与修复。

VchatManager is a **standalone Electron application** (with its own `package.json` and `main.js`) for managing VCPChat data, especially Agent/Group config consistency checking and repair.

### 文件 | Files

```
VchatManager/
├── main.js                        # Electron 主进程
├── preload.js                     # 预加载脚本
├── index.html                     # 主窗口 HTML
├── script.js                      # 渲染逻辑
├── style.css                      # 样式
├── consistency-checker.js         # 数据一致性检查器
├── package.json                   # 独立 Node.js 项目
├── start.bat                      # Windows 启动脚本
├── run_silent.vbs                 # 静默启动脚本
├── CONSISTENCY_CHECK_README.md    # 功能文档
└── FEATURE_SUMMARY.md             # 实现总结
```

### 核心功能 | Core Features

#### 数据一致性检查 | Data Consistency Check

```javascript
class ConsistencyChecker {
    // 检查所有 Agent 和 Group 的配置一致性
    async performCheck(agents, groups)

    // 检查单个项目（Agent 或 Group）
    async checkItem(itemId, itemData, itemType)

    // 修复发现的问题
    async fixIssues(selectedIssues, fixOptions)
}
```

**检测的问题类型 / Detected Issue Types:**

| 问题类型 | 描述 |
|---------|------|
| `orphaned_files` | 文件系统中存在话题目录，但 `config.json` 未记录 |
| `missing_files` | `config.json` 中有话题记录，但文件系统中目录不存在 |
| `missing_all_files` | 整个话题目录不存在 |

**修复选项 / Fix Options:**

- 添加孤立话题到配置（恢复遗失的历史）
- 从配置中移除不存在的话题引用（清理失效引用）

#### Agent 管理 | Agent Management

VchatManager 主界面提供：

- 浏览所有 Agent 列表（读取 `AppData/Agents/`）
- 浏览所有 Group 列表（读取 `AppData/AgentGroups/`）
- 查看 Agent/Group 配置
- 话题数量统计

### 启动方式 | Launch

```bash
cd VchatManager
npm install
npm start

# 或使用批处理文件
start.bat
```

---

## 3. `VCPHumanToolBox/` — 人类工具调用器 | Human Tool Invoker

### 概述 | Overview

VCPHumanToolBox 是另一个**独立的 Electron 应用**，为用户提供图形化界面直接调用 VCP 工具，无需通过 AI 中转。

VCPHumanToolBox is another **standalone Electron application** that provides a graphical interface for users to directly invoke VCP tools without going through AI.

### 文件 | Files

```
VCPHumanToolBox/
├── main.js                # Electron 主进程
├── preload.js             # 预加载脚本
├── index.html             # 主窗口 HTML
├── renderer.js            # 渲染逻辑
├── style.css              # 样式
├── package.json           # 独立 Node.js 项目
├── start.bat              # Windows 启动脚本
├── run_silent.vbs         # 静默启动脚本
└── VCHB.lnk              # Windows 快捷方式
```

### 核心功能 | Core Features

1. **自动 GUI 生成**: 自动读取 VCP 服务器上的插件列表，为每个插件生成图形化参数输入表单
2. **工作流引擎**: 可视化节点式工作流构建，支持多节点串联
3. **高级节点**:
   - 数据转换器（Data Transformer）
   - 高级条件判断（Advanced Conditionals）
   - 计时器/延时器（Timer/Delay）
   - 编辑器/循环节点（Editor/Loop）
4. **URL 渲染器**: 直接在窗口内渲染 PDF、音频、视频文件
5. **透明执行**: 调用过程和结果清晰展示，便于调试

### 与主应用的关系 | Relationship with Main App

VCPHumanToolBox 是完全独立的工具，但共享：
- 相同的 VCP 服务器 URL 和 API Key（读取 VCPChat 的 `AppData/settings.json` 或独立配置）
- 相同的 VCP 协议工具调用格式

### 启动方式 | Launch

```bash
cd VCPHumanToolBox
npm install
npm start
```

或通过 `VCHB.lnk` 快捷方式（Windows）。

---

## 4. 启动脚本 | Launch Scripts

项目根目录提供多种启动方式：

```
start.bat          # Windows 批处理启动（有控制台窗口）
启动Vchat.vbs      # VBS 脚本静默启动（无控制台窗口）
```

`启动Vchat.vbs` 内容：
```vbs
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run "npm start", 0, False
```

---

## 5. 工具间关系图 | Tool Relationship Diagram

```
VCPToolBox (后端服务器 / Backend Server)
    │
    ├──────────────────────────── VCPChat (主应用 / Main App)
    │   • 聊天界面 Chat UI           │
    │   • Agent 管理 Agent mgmt     │
    │                               ├── VCPDistributedServer (内嵌 / Embedded)
    │                               │   • 分布式节点 Distributed node
    │                               │   • 本地插件 Local plugins
    │                               │
    │                               ├── VchatManager (独立 / Standalone)
    │                               │   • 数据管理 Data management
    │                               │   • 一致性检查 Consistency check
    │                               │
    │                               └── VCPHumanToolBox (独立 / Standalone)
    │                                   • 工具 GUI GUI for tools
    │                                   • 工作流 Workflow engine
    │
    └── AI Model APIs (OpenAI-compatible endpoints)
```

---

*→ 下一章: [12_vendor_deps.md](./12_vendor_deps.md)*  
*← 上一章: [10_ui_themes_styles.md](./10_ui_themes_styles.md)*
