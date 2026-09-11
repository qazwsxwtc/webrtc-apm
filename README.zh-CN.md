> [English](README.md) | 中文

# WebRTC APM — 独立音频处理库

从 WebRTC 源码独立出来的 **Audio Processing Module**，仅保留音频相关功能，剔除视频/网络/ICE/媒体等无关模块，实现**单 CMake 工程、零外部依赖、跨平台**编译。

---

## 功能特性

| 模块 | 类 | 说明 |
|------|----|------|
| **HPF** | `AudioProcessing` 内置 | 高通滤波器，抑制低频噪声（默认 100Hz） |
| **AEC3** | `AudioProcessing` 内置 | 第三代回声消除器（基于频域自适应滤波） |
| **AECM** | `AudioProcessing` 内置 | 移动端回声消除（轻量级，Android/iOS 原生路径） |
| **NS** | `AudioProcessing` 内置 | 噪声抑制（时域 + 频域融合算法） |
| **NSX** | `AudioProcessing` 内置 | RNNoise 驱动的深度神经网络噪声抑制 |
| **AGC1** | `AudioProcessing` 内置 | 传统自动增益控制 |
| **AGC2** | `AudioProcessing` 内置 | 基于响度的第二代 AGC |
| **VAD** | `AudioProcessing` 内置 | 语音活动检测（GMM + 深度网络双引擎） |
| **HWA** | `AudioProcessing` 内置 | 硬件 AGC/Echo/Suppression 代理（可选） |

处理流水线：

```
Capture → HPF → AEC3/AECM → NS/NSX → AGC1/AGC2 → VAD → Output
```

---

## 构建要求

| 项目 | 最低版本 |
|------|---------|
| CMake | 3.16+ |
| C++ 编译器 | 支持 C++17（MSVC 2019+ / GCC 9+ / Clang 10+） |
| Python | 不需要 |
| GN/Ninja | 不需要 |

**零外部依赖** —— 所有源码（含 ISAC 编解码器、PFFFT、RNNoise、Abseil-CPP 子集）均已内置。

---

## 快速开始

### Windows (VS2022)

```powershell
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

### Linux / macOS

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
```

### Android (NDK)

```bash
cmake -S . -B build-android \
  -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24
cmake --build build-android -j
```

### iOS (Xcode)

```bash
cmake -S . -B build-ios \
  -G Xcode \
  -DCMAKE_OSX_SYSROOT=iphoneos \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build-ios --config Release
```

### Emscripten / WASM

```bash
emcmake cmake -S . -B build-wasm -DCMAKE_BUILD_TYPE=Release
cmake --build build-wasm -j
```

### 关闭示例程序

```bash
cmake -S . -B build -DAPM_BUILD_EXAMPLES=OFF
```

构建产物：

```
build/
├── Release/webrtc_apm.lib       # Windows
├── libwebrtc_apm.a              # Linux / macOS / Android / iOS
└── examples/                    # 7 个示例可执行文件
```

---

## 目录结构

```
apm/
├── CMakeLists.txt                   # 唯一构建入口
│
├── modules/
│   └── audio_processing/
│       ├── include/                 # ★ 对外主接口
│       │   ├── audio_processing.h          ← 创建实例 / 处理帧 / 读写配置
│       │   ├── config.h                    ← RuntimeConfig（开关各模块）
│       │   ├── audio_processing_statistics.h  ← getStatistics() 返回值
│       │   ├── audio_frame_proxies.h       ← Wrap 外部 AudioFrame
│       │   └── audio_frame_view.h          ← 视口类型
│       ├── aec3/                    # AEC3 核心实现
│       ├── aecm/                    # AECM 核心实现
│       ├── agc/                     # AGC1
│       ├── agc2/                    # AGC2
│       ├── ns/                      # NS + NSX
│       ├── vad/                     # VAD
│       ├── echo_detector/           # 回声检测（AEC3 辅助）
│       ├── transient/               # 瞬态抑制（已排除，仅 HPF/AEC/NS/AGC/VAD）
│       ├── utility/                 # 工具函数
│       └── aec_dump/                # null_aec_dump（无 IO 依赖）
│
├── modules/audio_coding/codecs/isac/   # ISAC 编解码器（VAD + AEC 依赖）
│   ├── main/{source,util}/
│   └── fix/source/
│
├── api/
│   ├── audio/                       # AudioFrame / EchoCanceller3Config / NoiseSuppression / GainControl
│   ├── task_queue/                  # TaskQueue 接口（默认 stdlib 实现）
│   ├── units/                       # Timestamp / TimeDelta / DataRate
│   ├── array_view.h                 # span 类
│   ├── audio_options.h
│   ├── function_view.h
│   ├── ref_counted_base.h
│   ├── scoped_refptr.h
│   └── rtp_headers.h / rtp_packet_info.h / rtp_packet_infos.h
│   ⚠ api/video/ 已删除 —— VideoRotation / VideoCodecType 等在 rtp_headers.h 内联占位
│
├── rtc_base/                        # 精简版基础库（9 子目录 / 242 files）
│   ├── system/                      # arch.h (自动检测 x86/ARM/MIPS) / cpu_features / ...
│   ├── synchronization/             # mutex / rw_lock (win + posix) / critical_section
│   ├── memory/                      # 内存工具
│   ├── task_queue*.cc               # win / stdlib / gcd / libevent 四套实现
│   ├── time_utils / clock / sleep   # 跨平台时间原语
│   ├── system_wrappers/             # field_trial / metrics / cpu_info
│   └── experiments/                 # 字段试验解析（video 相关已注释）
│
├── common_audio/                    # 信号处理 + AEC3/N 支撑
│   ├── signal_processing/           # SPL 库（resample / fft / filter）
│   ├── third_party/                 # spl_sqrt_floor / ooura_fft
│   └── ...
│
├── system_wrappers/                 # clock / field_trial / metrics / cpu_features
│
├── third_party/
│   ├── abseil-cpp/                  # Abseil 子集（9 目录 / 197 files）
│   │                                # 仅保留 base / algorithm / container /
│   │                                # flags / memory / meta / strings / types / utility
│   ├── pffft/                       # PFFFT 单精度 FFT（2 C 文件）
│   └── rnnoise/                     # RNNoise 神经网络 VAD（权重 + 推理）
│
└── example/                         # 7 个示例程序
    ├── basic_apm.cc                 # 最小编译 + 实例化
    ├── apm_pipeline.cc              # 完整流水线 HPF→AEC→NS→AGC→VAD
    ├── hpf_aec_ns_agc_vad.cc        # 同上，带统计输出
    ├── aec_demo.cc                  # 单独演示 AEC（ERLE 输出）
    ├── ns_demo.cc                   # 单独演示 NS（噪声衰减 dB）
    ├── agc_demo.cc                  # 单独演示 AGC
    └── vad_demo.cc                  # 单独演示 VAD（语音概率输出）
```

---

## 核心接口速览

```cpp
#include "modules/audio_processing/include/audio_processing.h"
#include "api/audio/audio_frame.h"

// 1. 创建实例
apm::AudioProcessingBuilder builder;
builder.SetCapturePostProcessing(std::make_unique<apm::EchoCanceller3Factory>());
auto apm = builder.Create();

// 2. 初始化：10ms/帧、16kHz、单声道、全双工
apm::AudioProcessing::Config cfg;
cfg.echo_canceller.enabled = true;
cfg.noise_suppression.enabled = true;
cfg.gain_controller1.enabled = true;
cfg.gain_controller2.enabled = false;
cfg.high_pass_filter.enabled = true;
apm->ApplyConfig(cfg);

apm->Initialize(16000, 16000, 1, 1, apm::AudioProcessing::kFullDuplex);

// 3. 处理每帧 10ms 音频
//    capture_frame: 麦克风数据
//    render_frame:  远端参考（AEC 必需）
apm->ProcessStream(capture_frame);
apm->ProcessReverseStream(render_frame);

// 4. 读统计
auto stats = apm->GetStatistics();
// stats.echo_cancel.erl_db      —— 回声返回损耗
// stats.noise_suppression.noise_level —— 噪声等级
// stats.voice_detected          —— VAD 结果

// 5. 关闭
apm.reset();
```

详细接口见 `modules/audio_processing/include/audio_processing.h`。

---

## 示例程序

编译后位于 `build/examples/` 或 `build-wasm/examples/`，无命令行参数，运行即生成合成音频并处理：

| 示例 | 输出指标 |
|------|---------|
| `basic_apm` | 实例创建 + 单帧处理成功标志 |
| `apm_pipeline` | HPF→AEC→NS→AGC→VAD 全流水线 |
| `hpf_aec_ns_agc_vad` | 同上 + ERLE / 噪声衰减 dB / VAD 概率 |
| `aec_demo` | ERLE (dB)，模拟回声消除效果 |
| `ns_demo` | 噪声衰减量 (dB) |
| `agc_demo` | 输入/输出电平对比 |
| `vad_demo` | 每帧语音概率 0.0–1.0 |

---

## 跨平台支持

| 平台 | 宏 | 链接库 | 源文件数 |
|------|----|--------|---------|
| **Windows** | `WEBRTC_WIN` + `NOMINMAX` | `winmm ws2_32` | APM=262 / RTC=42 |
| **Linux** | `WEBRTC_LINUX` + `WEBRTC_POSIX` | `pthread` | 同 Windows |
| **macOS** | `WEBRTC_APPLE` + `WEBRTC_OSX` + `WEBRTC_POSIX` | `pthread` | 同 Windows |
| **iOS** | `WEBRTC_APPLE` + `WEBRTC_IOS` + `WEBRTC_POSIX` | `pthread` | 同 Windows |
| **Android** | `WEBRTC_ANDROID` + `WEBRTC_POSIX` | `pthread log` | 用 `cpu_features_android.c`、`ifaddrs_android.cc` |
| **Emscripten** | `WEBRTC_LINUX` + `WEBRTC_POSIX` | (无) | 同 Linux |

架构自动检测（`rtc_base/system/arch.h`）：

- x86 / x86_64 → 启用 `_sse4_1` / `_avx2` 加速
- ARM32 / ARM64 → 启用 `_neon` 加速
- MIPS → 启用 `_mips` 加速

CMake 侧按 `CMAKE_SYSTEM_PROCESSOR` 自动过滤非目标架构的汇编文件。

---

## 精简历程

### 原始 WebRTC 源码 → 独立 APM

| 步骤 | 操作 | 效果 |
|------|------|------|
| 1 | 物理删除 36 个无关目录：`audio_mixer` / `api/video_codecs` / `api/video` / `call` / `media` / `rtc` / `java` / `video_coding` / `video_processing` / `rtp_rtcp` / `congestion_controller` / `pacing` / `remote_bitrate_estimator` ... | 砍掉 ~70% 顶层目录 |
| 2 | CMake 路径排除 `/transient/` / `/audio/utility/`（瞬态抑制已不需要） | 源文件从 351 → 262 |
| 3 | 删除所有 `.gn` / `.gni` / `.m` / BUILD.gn 文件 | 工程仅依赖 CMake |
| 4 | 手动排除 28 个无关 `.cc`：`*_factory` / `*_json` / `ssl_*` / `openssl_*` / `*_experiment` / `*_stats_counter` | 进一步收缩 |
| 5 | **Abseil-CPP 闭包裁剪**：18 子目录 1259 files → 9 子目录 197 files（删 time / random / debugging / flags / protocolbuffers ...） | 精简 84.4% |
| 6 | **rtc_base 闭包裁剪**：12 子目录 509 files → 9 子目录 242 files（删 java / network / time / win 平台裸文件 157 个） | 精简 52.5% |
| 7 | **删除 api/video/ 目录**（29 files）+ 注释 rtp_headers.h 中 video include + 注释 rtc_base/experiments 中 video 类型引用 | 彻底摆脱视频依赖 |

### 构建产物对比

| 指标 | 原始 WebRTC | 精简后 APM | 变化 |
|------|------------|-----------|------|
| 编译源文件 | ~1500+ | **304** | ↓ 80% |
| 目录数 | ~120 | **55** | ↓ 54% |
| webrtc_apm.lib | — | **7.36 MB** | — |
| 外部依赖 | GN / Ninja / Python / OpenSSL ... | **零** | — |

---

## 许可证

BSD 3-Clause License，与 WebRTC 原始代码一致。