# 09 — 数据存储结构 | Data Storage Structure

## 概述 | Overview

VCPChat 将所有运行时数据存储在项目根目录的 **`AppData/`** 文件夹中（与代码同目录，未加入 Git）。这种设计使备份和迁移非常简单：只需复制 `AppData/` 即可。

VCPChat stores all runtime data in the **`AppData/`** folder at the project root (co-located with code, not in Git). This design makes backup and migration trivial — just copy `AppData/`.

---

## 1. 完整目录结构 | Complete Directory Structure

```
AppData/
├── settings.json               # 全局应用设置 / Global app settings
├── songlist.json               # 音乐播放列表 / Music playlist
├── forum.config.json           # 论坛配置（凭据）/ Forum config (credentials)
├── network-notes-cache.json    # 网络笔记缓存 / Network notes cache
│
├── Agents/                     # Agent 配置目录 / Agent configs directory
│   └── {agentId}/              # 每个 Agent 一个子目录 (UUID 命名)
│       ├── config.json         # Agent 配置 / Agent configuration
│       └── avatar.png          # Agent 头像（可选）/ Agent avatar (optional)
│
├── AgentGroups/                # 群组配置目录 / Group configs directory
│   └── {groupId}/              # 每个群组一个子目录 (UUID 命名)
│       └── config.json         # 群组配置 / Group configuration
│
├── UserData/                   # 用户数据目录 / User data directory
│   ├── user_avatar.png         # 用户头像 / User avatar
│   ├── attachments/            # 中心化附件存储（内容寻址）
│   │   └── {sha256hash}{ext}   # 以 SHA-256 哈希命名的附件文件
│   │
│   ├── {agentId}/              # Agent 聊天数据（与 Agent 目录相同 ID）
│   │   └── topics/
│   │       └── {topicId}/      # 每个话题一个子目录 (UUID 命名)
│   │           └── history.json # 聊天历史 / Chat history
│   │
│   └── {groupId}/              # 群组聊天数据
│       └── topics/
│           └── {topicId}/
│               └── history.json
│
├── MusicCoverCache/            # 专辑封面缓存 / Album cover cache
│   └── {sha256hash}.{ext}
│
├── WallpaperThumbnailCache/    # 壁纸缩略图缓存 / Wallpaper thumbnail cache
│   └── {filename}_thumb.jpg
│
├── ResampleCache/              # 音频重采样缓存 / Audio resample cache
│   └── {hash}_{samplerate}.pcm
│
├── canvas/                     # Canvas 编辑器文件 / Canvas editor files
│   ├── {filename}              # 文件内容 / File content
│   └── .history/
│       └── {filename}/
│           └── {timestamp}.json # 版本历史快照 / Version history snapshots
│
└── Notemodules/                # 笔记模块数据 / Notes module data
    └── ...
```

---

## 2. 关键数据文件格式 | Key Data File Formats

### 2.1 `settings.json` — 全局设置

```json
{
    "vcpServerUrl": "http://localhost:3000/v1/chat/completions",
    "vcpApiKey": "your-api-key",
    "vcpLogUrl": "ws://localhost:5890",
    "vcpLogKey": "your-log-key",
    "userName": "用户",
    "theme": "dark",
    "wallpaper": "",
    "sidebarWidth": 260,
    "notificationsSidebarWidth": 300,
    "topicSummaryModel": "gemini-2.5-flash",
    "enableThoughtChainInjection": false,
    "flowlockContinueDelay": 5,
    "filterEnabled": false,
    "filterRules": [],
    "enableRegenerateConfirmation": true,
    "enableMiddleClickQuickAction": false,
    "middleClickQuickAction": "",
    "ttsGlobalEnabled": false,
    "sovitsServerUrl": "",
    "hotModels": [],
    "favoriteModels": []
}
```

### 2.2 `Agents/{agentId}/config.json` — Agent 配置

```json
{
    "name": "助手",
    "model": "gpt-4o",
    "temperature": 0.7,
    "contextTokenLimit": 128000,
    "maxOutputTokens": 32000,
    "topP": 1.0,
    "topK": 0,
    "systemPrompt": "",
    "originalSystemPrompt": "",
    "advancedSystemPrompt": [],
    "presetSystemPrompt": "",
    "promptMode": "original",
    "vcpServerUrl": "",
    "avatarBorderColor": "",
    "nameTextColor": "",
    "customCss": "",
    "ttsVoicePrimary": "",
    "ttsRegexPrimary": "",
    "ttsVoiceSecondary": "",
    "ttsRegexSecondary": "",
    "ttsSpeed": 1.0,
    "regexRules": [],
    "topics": [
        {
            "id": "uuid",
            "title": "新对话",
            "createdAt": "2025-01-01T00:00:00.000Z",
            "updatedAt": "2025-01-01T00:00:00.000Z",
            "locked": false
        }
    ],
    "tags": [],
    "preset": null,
    "worldBook": null
}
```

### 2.3 `AgentGroups/{groupId}/config.json` — 群组配置

```json
{
    "name": "群组名称",
    "members": [
        {
            "agentId": "uuid1",
            "tags": ["tag1", "tag2"]
        },
        {
            "agentId": "uuid2",
            "tags": []
        }
    ],
    "speechMode": "sequential",
    "groupPrompt": "群聊背景设定...",
    "invitePrompt": "现在轮到你{{VCPChatAgentName}}发言了...",
    "topics": [
        {
            "id": "uuid",
            "title": "主要群聊",
            "createdAt": "...",
            "updatedAt": "..."
        }
    ]
}
```

### 2.4 `UserData/{agentId}/topics/{topicId}/history.json` — 聊天历史

```json
[
    {
        "id": "msg-uuid",
        "role": "user",
        "content": "Hello!",
        "timestamp": "2025-01-01T00:00:00.000Z",
        "attachments": [
            {
                "internalPath": "AppData/UserData/attachments/abc123.png",
                "originalName": "image.png",
                "mimeType": "image/png",
                "hash": "abc123..."
            }
        ]
    },
    {
        "id": "msg-uuid2",
        "role": "assistant",
        "content": "Hi there!",
        "timestamp": "2025-01-01T00:00:01.000Z",
        "model": "gpt-4o",
        "attachments": []
    }
]
```

### 2.5 正则规则格式 | Regex Rule Format

存储在 `config.json` 的 `regexRules` 数组中：

```json
[
    {
        "id": "rule-uuid",
        "name": "规则名称",
        "enabled": true,
        "findPattern": "/pattern/gi",
        "replaceWith": "replacement",
        "scope": "frontend",
        "roles": ["user", "assistant"],
        "depthMin": 0,
        "depthMax": -1
    }
]
```

**scope 值 / Scope values:**
- `frontend` — 只影响前端显示，不修改历史
- `context` — 修改发送给 AI 的上下文（历史压缩时）
- `renderer` — 渲染时应用
- `deep` — 深度正则
- `content_array` — content 数组格式的正则

---

## 3. 内容寻址附件存储 | Content-Addressed Attachment Storage

所有用户上传的附件（图片、文档等）统一存储在：

```
AppData/UserData/attachments/{sha256_hash}{original_extension}
```

**优点 / Benefits:**

1. **去重**: 相同内容只存一份（通过 SHA-256 哈希检测）
2. **稳定引用**: 即使原文件移动，内部 URL 不变
3. **跨话题共享**: 多个话题引用同一文件不会重复存储

**访问方式 / Access method:**

通过自定义协议 `vcpchat-file://` 访问：

```javascript
// 注册于 main.js
protocol.registerFileProtocol('vcpchat-file', (request, callback) => {
    const filePath = decodeURIComponent(request.url.slice('vcpchat-file://'.length))
    callback({ path: path.normalize(filePath) })
})
```

---

## 4. Agent 数据一致性检查 | Agent Data Consistency Check

`VchatManager/consistency-checker.js` 提供数据一致性检查工具，检测：

- **orphaned_files**: 文件系统存在但 `config.json` 不知道的话题目录
- **missing_files**: `config.json` 记录但文件系统中缺失的话题目录
- **missing_all_files**: 话题目录完全不存在

可通过 VchatManager 工具（→ 见 [11_companion_tools.md](./11_companion_tools.md)）运行修复。

---

## 5. 数据迁移 | Data Migration

`migration/` 目录包含数据迁移脚本：

```
migration/
├── migrateAvatars.js     # 头像存储路径迁移脚本
└── 头像迁移脚本readme.md  # 迁移说明
```

当数据格式发生变化时（如早期版本到新版本升级），使用这些脚本进行一次性数据迁移。

---

## 6. 备份脚本 | Backup Script

`backup.py` 是一个 Python 备份脚本，可将 `AppData/` 目录打包备份。

---

*→ 下一章: [10_ui_themes_styles.md](./10_ui_themes_styles.md)*  
*← 上一章: [08_vcp_protocol.md](./08_vcp_protocol.md)*
