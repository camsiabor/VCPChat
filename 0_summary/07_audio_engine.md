# 07 — 音频引擎 | Audio Engine

## 概述 | Overview

VCPChat 的音频引擎经历了从 Python → Rust 的迁移，目前以 **Rust 原生二进制**（`audio_server`）为核心，提供 Hi-Fi 级别的音乐播放能力。

VCPChat's audio engine migrated from Python to **Rust native binary** (`audio_server`), providing Hi-Fi-grade music playback capability.

---

## 1. 架构概览 | Architecture Overview

```
┌────────────────────────────────────────────────────┐
│  Electron Main Process (main.js)                   │
│  startAudioEngine() → spawn audio_engine/audio_server │
└──────────────────────┬─────────────────────────────┘
                       │  HTTP REST API
                       │  localhost:63789
┌──────────────────────▼─────────────────────────────┐
│  Rust Audio Server (audio_engine/audio_server[.exe]) │
│  • 音频解码 / Audio decoding                        │
│  • 重采样 (SoXR) / Resampling (SoXR)               │
│  • WASAPI 独占模式 / WASAPI exclusive mode          │
│  • 均衡器 (FIR/IIR EQ)                             │
│  • 升频处理 / Upsampling                            │
│  • 磁盘缓存 / Disk cache                           │
└──────────────────────────────────────────────────────┘
                       ▲
                       │  IPC (musicHandlers.js)
┌──────────────────────┴─────────────────────────────┐
│  Renderer Process (Musicmodules/music.js)           │
│  • 播放列表 UI / Playlist UI                       │
│  • 歌词显示 / Lyrics display                       │
│  • EQ 控制界面 / EQ control UI                     │
└────────────────────────────────────────────────────┘
```

---

## 2. `audio_engine/` — 预编译二进制 | Pre-built Binary

### 目录内容 / Contents

```
audio_engine/
├── audio_server            # Linux/macOS 二进制
├── audio_server.exe         # Windows 二进制 (AVX2 优化版)
├── audio_server 通用CPU兼容版.exe  # Windows 二进制 (通用 CPU 版本)
└── .env                    # 音频引擎配置文件
```

### 配置文件 `.env` | Configuration File

```env
VCP_AUDIO_EQ_TYPE=FIR                  # 默认 EQ 类型: FIR 或 IIR
VCP_AUDIO_TARGET_SAMPLERATE=192000     # 目标升频采样率 (Hz)
VCP_AUDIO_USE_CACHE=true               # 启用重采样磁盘缓存
VCP_AUDIO_PREEMPTIVE_RESAMPLE=true     # 抢跑重采样 (非独占模式)
VCP_AUDIO_RESAMPLE_QUALITY=hq          # 重采样质量: low/std/hq/uhq
```

### 启动参数 / Launch Arguments

```bash
audio_server --port 63789
```

主进程等待 `RUST_AUDIO_ENGINE_READY` 标准输出信号（最多等待 10 秒）。

### REST API 端点 | REST API Endpoints

音频引擎通过本地 HTTP 服务暴露以下端点（由 `musicHandlers.js` 调用）：

```
POST /load              加载音频文件
POST /play              播放
POST /pause             暂停
POST /seek              跳转到指定位置
GET  /state             获取当前播放状态
POST /volume            设置音量
GET  /devices           获取音频输出设备列表
POST /configure         配置输出设备 (WASAPI 设备选择)
POST /eq                设置均衡器参数
POST /eq-type           切换 EQ 类型 (FIR/IIR)
POST /optimizations     配置音质优化选项
POST /upsampling        配置升频设置
```

---

## 3. `rust_audio_engine/` — Rust 源码 | Rust Source Code

### 目录内容 / Contents

```
rust_audio_engine/
├── Cargo.toml            # Rust 项目配置
├── Cargo.lock            # 锁定依赖版本
├── soxr.pc               # SoXR 库 pkg-config 配置
├── README.md             # 构建说明
└── 前置工作.md           # 依赖安装说明 (中文)
```

### 关键技术特性 / Key Technical Features

| 特性 Feature | 说明 Description |
|-------------|-----------------|
| **SIMD 加速** | 通过 `RUSTFLAGS=-C target-cpu=native` 启用 CPU 特定优化 |
| **Python 3.13 支持** | 使用 maturin 构建 Python 扩展模块（可选安装）|
| **64-bit Pipeline** | 内部全程双精度浮点处理 |
| **高精度相位时钟** | `QualityFlags::HighPrecisionClock` 提升无理数采样率比精度 |
| **极高品质重采样** | `QualityRecipe::very_high()` (Bits28) |
| **多通道支持** | 1-2 通道用 Stereo 格式，3+ 通道逐通道处理 |
| **SoXR 库** | 使用 libsoxr 进行专业级音频重采样 |

### 构建方式 / Build Instructions

```bash
# Windows (CMD) - 需要预先通过 vcpkg 安装 soxr
cd rust_audio_engine
set PATH=<vcpkg>/installed/x64-windows-static/tools/pkgconf;%PATH%
set PKG_CONFIG_PATH=<vcpkg>/installed/x64-windows-static/lib/pkgconfig
set RUSTFLAGS=-C target-cpu=native

# 构建 audio_server 二进制
cargo build --release

# 构建 Python 扩展（可选）
py -3.13 -m maturin build --release --interpreter python3.13
```

输出文件：`target/release/audio_server[.exe]`

---

## 4. Python 重采样扩展（可选）| Python Resampler Extension (Optional)

在早期版本或某些配置中，项目还支持 Python 音频处理。`requirements.txt` 中包含音频相关依赖，`pyproject.toml` 定义了 Python 项目结构。

在 64 位 Windows + Python 3.13 环境中，可直接安装预编译的 Rust 扩展：

```bash
pip install audio_engine/rust_audio_resampler-0.1.0-cp313-cp313-win_amd64.whl
```

---

## 5. 进程生命周期 | Process Lifecycle

```
Electron 启动
    │
    ▼
main.js: startAudioEngine()
    │  spawn audio_server --port 63789
    ▼
等待 stdout: "RUST_AUDIO_ENGINE_READY"
    │  (超时 10 秒则 reject)
    ▼
音频引擎就绪 → 主窗口可以使用音乐功能
    │
    ▼
Electron 退出
    │
    ▼
main.js: stopAudioEngine()
    │  audioEngineProcess.kill()
    ▼
音频引擎进程终止
```

---

## 6. SovitsTest/ — SoVITS TTS 测试 | SoVITS TTS Testing

```
SovitsTest/
├── GSVI.py           # GPT-SoVITS 推理接口测试
├── get_models.py     # 获取可用模型列表
├── my_infer.py       # 自定义推理脚本
├── test_sovits_api.py # API 测试脚本
├── output.wav        # 测试输出音频
└── README.md         # SoVITS 集成说明
```

SoVITS TTS 是可选的语音合成功能，通过 HTTP API 与独立的 SoVITS 服务通信。主聊天流中的 `modules/SovitsTTS.js` 封装了该 API 调用。

---

*→ 下一章: [08_vcp_protocol.md](./08_vcp_protocol.md)*  
*← 上一章: [06_feature_modules.md](./06_feature_modules.md)*
